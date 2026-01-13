# Protocole CPE (Crypté & Protégé pour l'Envoi)

## Vue d'ensemble

Le protocole CPE est un protocole de communication sécurisé pour la transmission de données des véhicules et incidents via radio. Il utilise le chiffrement AES-128 en mode CTR et assure l'intégrité des données par CRC.

## Caractéristiques techniques

### Tailles de trame
- **Taille totale** : 30 octets
- **En-tête** : 10 octets (seq + nonce + timestamp + crc8)
- **Payload chiffré** : 18 octets (données) + 2 octets (CRC16)
- **Clé de chiffrement** : 16 octets (AES-128)
- **Immatriculation** : 8 octets

### Types de trames
| Type | Code | Description |
|------|------|-------------|
| `VEH_POS` | 0x01 | Position et statut d'un véhicule |
| `VEH_STATUS` | 0x02 | Statut seul d'un véhicule |
| `INCIDENT_AFFECT` | 0x03 | Affectation à un incident |
| `INCIDENT_STATUS` | 0x04 | Statut d'un incident |

## Structure de la trame

### En-tête (10 octets, en clair)
```
[SEQ:1] [NONCE:4] [TIMESTAMP:4] [CRC8:1]
```
- **SEQ** : Numéro de séquence (1-255, jamais 0)
- **NONCE** : Nombre aléatoire 32-bit (pour l'IV du chiffrement)
- **TIMESTAMP** : Horodatage Unix 32-bit
- **CRC8** : Contrôle d'intégrité de l'en-tête (polynôme 0x31)

### Payload (20 octets, chiffré)
```
[VER/TYPE:1] [IMMAT:8] [STATUS:1] [LAT:4] [LON:4] [CRC16:2]
```
- **VER/TYPE** : Version (4 bits hauts) + Type de trame (4 bits bas)
- **IMMAT** : Immatriculation du véhicule (8 caractères)
- **STATUS** : Code de statut (selon le type de trame)
- **LAT/LON** : Coordonnées GPS en micro-degrés (E6, int32)
- **CRC16** : Contrôle d'intégrité du payload (CRC-CCITT)

## Chiffrement

**Algorithme** : AES-128-CTR

**Vecteur d'initialisation (IV)** :
```
IV = [NONCE:4] [SEQ:1] [00:11]
```
Le nonce et le numéro de séquence forment les 5 premiers octets du compteur CTR.

## Sécurité

- **Authentification** : Via CRC double (header + payload)
- **Confidentialité** : Chiffrement AES-128 du payload
- **Protection contre le replay** : Nonce et séquence uniques
- **Intégrité** : CRC8 sur l'en-tête, CRC16 sur le payload

## API C

### Initialisation
```c
void cpe_init(const uint8_t key[16]);
```

### Construction de trames
```c
void cpe_build_vehicle_position_frame(
    const char immat[8],
    int32_t lat_e6,
    int32_t lon_e6,
    uint8_t status,
    uint32_t nonce,
    uint32_t timestamp,
    uint8_t seq,
    uint8_t out_frame[30]
);

void cpe_build_vehicle_status_frame(...);
void cpe_build_incident_affect_frame(...);
void cpe_build_incident_status_frame(...);
```

### Parsing
```c
int cpe_parse_frame(
    const uint8_t frame[30],
    cpe_frame_type_t *type_out,
    char immat_out[8],
    uint8_t *status_out,
    int32_t *lat_e6_out,
    int32_t *lon_e6_out,
    uint32_t *nonce_out,
    uint32_t *timestamp_out,
    uint8_t *seq_out
);
```
Retourne 0 si succès, -1 si erreur (CRC invalide ou format incorrect).

## Notes d'implémentation

- Les numéros de séquence commencent à 1 et se réinitialisent après 255
- Les coordonnées GPS sont stockées en micro-degrés (lat/lon * 1 000 000)
- Le CRC8 détecte les corruptions radio sur l'en-tête
- Le CRC16 valide l'intégrité du payload après déchiffrement
- Tous les entiers multi-octets sont en big-endian
