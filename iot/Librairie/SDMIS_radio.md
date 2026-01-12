# Librairie SDMIS_RADIO

## Vue d'ensemble

La librairie SDMIS_RADIO est une couche d'abstraction pour la communication radio sur micro:bit. Elle encapsule le protocole CPE et fournit une API simple pour échanger des messages sécurisés entre unités.

## Fonctionnalités

- **Fiabilité** : Mécanisme d'acquittement (ACK) avec retransmissions automatiques
- **Sécurité** : Chiffrement AES-128 via le protocole CPE
- **Gestion automatique** : Nonce, séquence, déduplication des messages
- **Buffer de réception** : File circulaire pour stocker jusqu'à 4 trames reçues
- **Drain UART** : Callback pour éviter l'overflow pendant les opérations radio bloquantes

## Architecture

```
Application micro:bit
        ↓
  SDMIS_RADIO (ACK/Retry)
        ↓
  Protocole CPE (Crypto)
        ↓
  MicroBit Radio (2.4 GHz)
```

## Structure de données

### `sdmis_frame_t`
```cpp
typedef struct {
    cpe_frame_type_t type;      // Type de trame
    char immat[9];              // Immatriculation (8+1 pour '\0')
    uint8_t status;             // Statut
    int32_t lat_e6;             // Latitude en micro-degrés
    int32_t lon_e6;             // Longitude en micro-degrés
    uint32_t nonce;             // Nonce de la trame
    uint32_t timestamp;         // Horodatage
    uint8_t seq;                // Numéro de séquence
} sdmis_frame_t;
```

## API

### Initialisation
```cpp
void sdmis_radio_init(
    MicroBit *uBit,
    const uint8_t key[16],
    uint8_t radio_group,
    uint8_t tx_power,
    sdmis_drain_callback_t drain_cb
);
```
- **uBit** : Pointeur vers l'instance MicroBit
- **key** : Clé AES-128 partagée
- **radio_group** : Groupe radio (0-255)
- **tx_power** : Puissance d'émission (0-7)
- **drain_cb** : Callback pour vider l'UART pendant les attentes (peut être NULL)

### Réception
```cpp
bool sdmis_radio_poll(sdmis_frame_t *out_frame);
```
Récupère la prochaine trame du buffer de réception. Retourne `true` si une trame est disponible.

**Usage** :
```cpp
sdmis_frame_t frame;
if (sdmis_radio_poll(&frame)) {
    // Traiter la trame reçue
}
```

### Émission

#### Position véhicule
```cpp
bool sdmis_radio_send_vehicle_position(
    const char immat[8],
    int32_t lat_e6,
    int32_t lon_e6,
    uint8_t status,
    uint32_t timestamp
);
```

#### Statut véhicule
```cpp
bool sdmis_radio_send_vehicle_status(
    const char immat[8],
    uint8_t status,
    uint32_t timestamp
);
```

#### Affectation incident
```cpp
bool sdmis_radio_send_incident_affect(
    const char immat[8],
    int32_t lat_e6,
    int32_t lon_e6,
    uint32_t timestamp
);
```

#### Statut incident
```cpp
bool sdmis_radio_send_incident_status(
    const char immat[8],
    uint8_t status,
    uint32_t timestamp
);
```

Toutes les fonctions d'envoi retournent `true` si l'ACK a été reçu, `false` sinon.

## Mécanisme de fiabilité

### Envoi avec retransmissions
- **Tentatives** : 3 essais maximum
- **Timeout ACK** : 200 ms
- **Backoff aléatoire** : 10-40 ms entre les tentatives

### Réception
- **ACK automatique** : Envoyé immédiatement à la réception d'une trame valide
- **Déduplication** : Basée sur (seq, nonce) pour éviter les doublons
- **Buffer circulaire** : 4 slots pour lisser les pics de trafic

## Paramètres de configuration

```cpp
#define SDMIS_ACK_TIMEOUT_MS 200
#define SDMIS_RETRY_COUNT 3
#define SDMIS_POLL_SLICE_MS 5
#define SDMIS_POLL_MAX 8
#define SDMIS_RX_SLOTS 4
```

## Exemple d'utilisation

```cpp
#include "lib/sdmis_radio.h"

MicroBit uBit;
uint8_t aes_key[16] = {...};

void drain_uart() {
    // Lire les données UART pour éviter l'overflow
}

int main() {
    uBit.init();
    
    // Initialisation
    sdmis_radio_init(&uBit, aes_key, 42, 6, drain_uart);
    
    while(1) {
        // Envoi
        bool ack = sdmis_radio_send_vehicle_position(
            "VEH12345", 
            48858859, // Paris
            2294481,
            1,        // En route
            uBit.systemTime()
        );
        
        // Réception
        sdmis_frame_t frame;
        if (sdmis_radio_poll(&frame)) {
            // Traiter frame
        }
        
        uBit.sleep(1000);
    }
}
```

## Notes importantes

- La librairie gère automatiquement les numéros de séquence (1-255 en boucle)
- Les nonces sont générés aléatoirement à chaque trame
- Le callback `drain_cb` est appelé pendant les attentes d'ACK pour éviter les blocages UART
- La déduplication protège contre la réception multiple de la même trame
- Le buffer circulaire permet de gérer les rafales sans perte
