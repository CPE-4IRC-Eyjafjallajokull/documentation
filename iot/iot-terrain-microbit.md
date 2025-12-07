# App Terrain micro:bit (émetteur)

micro:bit embarqué dans le véhicule pour émettre les informations terrain par radio (positions, états, messages opérateur).

## Rôle
- Envoi périodique GPS + statut (dispo/en route/sur intervention).
- Envoi manuel : arrivée sur site, fin d'intervention, demande de renfort, message libre.
- Respect du protocole RF/UART décrit dans la section IoT (trames sécurisées + identifiants véhicule).

## Build
Prérequis : GNU Arm Embedded Toolchain (`arm-none-eabi-*`), CMake 3.6+, Python 3.

```bash
cd iot-terrain-microbit
python3 build.py
# Génère MICROBIT.hex et MICROBIT.bin à la racine
```

## Arborescence
- `source/main.cpp` : point d'entrée (scheduler CODAL) à enrichir.
- `source/proto/`, `source/crypto/` : helpers de trames/AES-CMAC (doivent rester cohérents avec la passerelle centrale).

## Points à détailler
- Mapping des boutons/inputs vers les messages envoyés.
- Gestion des pertes radio (retries, ack éventuel) et cadence d'envoi GPS.
- Configuration des clés/identifiants par véhicule.
