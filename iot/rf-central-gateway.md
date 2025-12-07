# Passerelle RF Centrale (micro:bit)

Récepteur radio côté QG : capte les trames émises par les véhicules, les valide et les retransmet vers le moteur Java via UART.

## Rôle
- Balayage RF et réception de toutes les trames véhicules.
- Validation minimale (intégrité, horodatage) et éventuelle sécurisation (AES/CMAC TinyCrypt).
- Transmission UART vers le moteur ou une machine hôte pour ingestion RabbitMQ/API.

## Build
Prérequis : GNU Arm Embedded Toolchain (`arm-none-eabi-*`), CMake 3.6+, Python 3.

```bash
cd rf-central-gateway
python3 build.py
# Génère MICROBIT.hex et MICROBIT.bin à la racine
```

## Arborescence
- `source/main.cpp` : point d'entrée CODAL à enrichir pour la passerelle.
- `source/proto/`, `source/crypto/` : helpers de trames/AES-CMAC (alignés avec l'app terrain).

## Points à détailler
- Paramètres radio (canal, puissance) et format UART attendu côté moteur.
- Gestion des collisions et de la perte de messages (buffers, replays éventuels).
- Journalisation minimale pour débogage (LED/USB série).
