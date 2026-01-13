# Micro:bit terrain (iot-terrain-microbit)

Le micro:bit terrain est embarque dans les vehicules. Il envoie les positions
et statuts, et recoit les affectations.

---

## Rôle

- Envoi periodique de la position GPS.
- Envoi des changements de statut vehicule.
- Reception des affectations et messages terrain.

---

## Build et flash (Docker)

```bash
cd iot-terrain-microbit
make build
# Copiez ensuite le HEX genere sur le disque MICROBIT
```

HEX genere : `out/iot-terrain-microbit.hex`

---

## Build local (yotta)

```bash
yt clean
yt build
```

HEX local : `build/bbc-microbit-classic-gcc/source/microbit-samples-combined.hex`

---

## Flash avec Makefile

```bash
make yotta-build
make install
```

Si votre micro:bit est monte ailleurs :

```bash
make install MICROBIT_MOUNT=/Volumes/MICROBIT/
```

---

## Documentation associee

- Vue IoT globale : [README.md](README.md)
- Librairie SDMIS Radio : [librairie/sdmis-radio.md](librairie/sdmis-radio.md)
- Protocole CPE : [protocole/protocole-cpe.md](protocole/protocole-cpe.md)
