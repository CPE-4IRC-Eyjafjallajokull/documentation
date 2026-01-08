# 📚 Index des Documentations - Système SDMIS IoT Terrain

Documentation complète du système de communication radio sécurisée pour véhicules d'intervention.

---

## 📖 Guides principaux

### 🎯 [Vue d'ensemble du système](Systeme_complet/Systeme_complet.md)
**Pour qui** : Chef de projet, architecte, développeur débutant  
**Contenu** : Architecture globale, flux de communication, déploiement complet  
**⏱️ Lecture** : 20-30 minutes

Comprend :
- Architecture système complète
- Tous les composants et leurs interactions
- Guide de déploiement de A à Z
- Tests et validation
- Performances et limitations

---

## 🔧 Documentation des composants

### 📡 [Passerelle UART ↔ Radio](Passerelles/Passerelle_UART_radio.md)
**Pour qui** : Développeur, intégrateur  
**Contenu** : Passerelle bidirectionnelle entre simulateur Java et réseau radio  
**⏱️ Lecture** : 15 minutes

Comprend :
- Configuration UART (115200 bps)
- Format CSV des messages
- Mécanisme ACK/Retry
- Déduplication automatique
- Indicateurs visuels (T/!/A)
- Code source principal

---

### 📱 [Application Terrain (Émetteur)](Applications/App_terrain.md)
**Pour qui** : Développeur embarqué  
**Contenu** : Carte micro:bit embarquée dans les véhicules  
**⏱️ Lecture** : 15 minutes

Comprend :
- Envoi périodique de positions GPS
- Gestion boutons (statuts, événements)
- Réception affectations d'incidents
- Intégration module GPS
- Code source type
- Personnalisation par véhicule

---

### 🏢 [Passerelle RF Centrale](Passerelles/Passerelle_RF_centrale.md)
**Pour qui** : Développeur backend, DevOps  
**Contenu** : Récepteur central vers API backend  
**⏱️ Lecture** : 10 minutes

Comprend :
- Firmware micro:bit récepteur
- Gateway Python (UART → API)
- Configuration SSE pour affectations
- Déploiement backend
- Variables d'environnement

---

## 🔐 Documentation technique protocolaire

### 🔒 [Protocole CPE](Protocole/Protocole_CPE.md)
**Pour qui** : Développeur système, cryptographe  
**Contenu** : Spécification complète du protocole de chiffrement  
**⏱️ Lecture** : 25 minutes

Comprend :
- Structure trame 29 octets
- Chiffrement AES-128 CTR
- Calcul CRC-16 CCITT
- 3 types de messages (Position, Statut, Incident)
- API C complète
- Vecteur d'initialisation
- Recommandations sécurité

---

### 📻 [Librairie SDMIS_RADIO](Librairie/SDMIS_radio.md)
**Pour qui** : Développeur micro:bit  
**Contenu** : API haut niveau pour communication radio fiable  
**⏱️ Lecture** : 20 minutes

Comprend :
- Architecture logicielle (3 couches)
- Mécanisme ACK/Retry (3 tentatives)
- Anti-duplication (seq + nonce)
- API simple : init(), poll(), send_*()
- Exemples complets (émetteur/récepteur)
- Performances et limitations
- Dépannage

---

## 🚀 Guides pratiques

### ⚡ Guide de démarrage rapide

**Objectif** : Système fonctionnel en 30 minutes

1. **Cloner le projet**
   ```bash
   git clone https://github.com/votre-org/iot-terrain-microbit.git
   cd iot-terrain-microbit
   ```

2. **Compiler les firmwares**
   ```bash
   make clean && make build
   # Génère: out/iot-terrain-microbit.hex
   ```

4. **Flasher les cartes**
   - Passerelle : Flash sur micro:bit #1
   - Terrain : Flash sur micro:bit #2
   
5. **Lancer le simulateur Java**
   - Port série : `/dev/ttyACM0` (Linux) ou `COM4` (Windows)
   - Baudrate : 115200

6. **Tester**
   - Envoyer : `vehicle_position,TEST001,48.856614,2.352222,1736172600`
   - Observer : "T" sur passerelle, "✓" sur terrain

---

### 🔧 Guide de dépannage

| Problème | Voir documentation | Section |
|----------|-------------------|---------|
| Aucune communication radio | [SYSTEME_COMPLET](Systeme_complet/Systeme_complet.md) | Diagnostic problèmes |
| ACK non reçus | [SDMIS_RADIO](Librairie/SDMIS_radio.md) | Dépannage |
| Erreur compilation | [APP_TERRAIN](Applications/App_terrain.md) | Compilation |
| Messages CSV invalides | [PASSERELLE_UART_RADIO](Passerelles/Passerelle_UART_radio.md) | Format données |
| Clé cryptographique | [PROTOCOLE_CPE](Protocole/Protocole_CPE.md) | Sécurité |

---

## 📊 Tableaux de référence

### Configuration radio

| Paramètre | Valeur | Fichier config |
|-----------|--------|----------------|
| Groupe radio | 42 | `source/main.cpp` |
| Puissance TX | 7 (max) | `source/main.cpp` |
| Fréquence | 2.4 GHz | (matériel) |
| Portée | 100-150 m | - |

### Format des trames

| Type | Taille | Chiffrement | Documentation |
|------|--------|-------------|---------------|
| CPE | 29 octets | AES-128 CTR | [PROTOCOLE_CPE](Protocole/Protocole_CPE.md) |
| ACK | 1 octet | Non | [SDMIS_RADIO](Librairie/SDMIS_radio.md) |
| CSV UART | ~60 octets | Non | [PASSERELLE_UART_RADIO](Passerelles/Passerelle_UART_radio.md) |

### Latences typiques

| Opération | Latence | Documentation |
|-----------|---------|---------------|
| UART → Radio (succès) | 30-50 ms | [PASSERELLE_UART_RADIO](Passerelles/Passerelle_UART_radio.md) |
| UART → Radio (échec 3×) | ~650 ms | [PASSERELLE_UART_RADIO](Passerelles/Passerelle_UART_radio.md) |
| Radio → UART | 10-20 ms | [PASSERELLE_UART_RADIO](Passerelles/Passerelle_UART_radio.md) |
| Bouton → TX | 20-50 ms | [APP_TERRAIN](Applications/App_terrain.md) |

---

## 🎓 Parcours de lecture recommandés

### Pour débuter (Nouveau développeur)

1. ⭐ [Vue d'ensemble du système](Systeme_complet/Systeme_complet.md) - Comprendre l'architecture
2. ⭐ [Passerelle UART-Radio](Passerelles/Passerelle_UART_radio.md) - Commencer par le composant central
3. [Librairie SDMIS_RADIO](Librairie/SDMIS_radio.md) - Comprendre l'API
4. [Guide démarrage rapide](#-guide-de-démarrage-rapide) - Mise en pratique

### Pour développer (Contributeur)

1. [Protocole CPE](Protocole/Protocole_CPE.md) - Comprendre la couche crypto
2. [Librairie SDMIS_RADIO](Librairie/SDMIS_radio.md) - Comprendre la couche fiabilité
3. [Application Terrain](Applications/App_terrain.md) - Voir cas d'usage complet
4. Code source dans `source/`

### Pour déployer (Ops/Intégrateur)

1. [Vue d'ensemble](Systeme_complet/Systeme_complet.md) - Section "Installation et déploiement"
2. [Passerelle UART-Radio](Passerelles/Passerelle_UART_radio.md) - Section "Déploiement"
3. [Application Terrain](Applications/App_terrain.md) - Section "Compilation et déploiement"
4. [Passerelle RF Centrale](Passerelles/Passerelle_RF_centrale.md) - Si backend utilisé

### Pour sécuriser (RSSI/Auditeur)

1. [Protocole CPE](Protocole/Protocole_CPE.md) - Section "Sécurité"
2. [Vue d'ensemble](Systeme_complet/Systeme_complet.md) - Section "Sécurité du système"
3. [Passerelle UART-Radio](Passerelles/Passerelle_UART_radio.md) - Section "Sécurité et fiabilité"
4. Recommandations de rotation de clés

---

## 📦 Structure du projet

```
iot-terrain-microbit/
├── docs/                           # 📚 Toute la documentation
│   ├── README.md                   # ⭐ Ce fichier (index)
│   ├── SYSTEME_COMPLET.md          # Vue d'ensemble globale
│   ├── PROTOCOLE_CPE.md            # Spéc protocole chiffrement
│   ├── SDMIS_RADIO.md              # API librairie radio
│   ├── PASSERELLE_UART_RADIO.md    # Passerelle Java ↔ Radio
│   ├── APP_TERRAIN.md              # Application embarquée
│   └── PASSERELLE_RF_CENTRALE.md   # Gateway backend (optionnel)
├── source/                         # Code source
│   ├── main.cpp                    # Point d'entrée
│   ├── lib/                        # Librairie SDMIS_RADIO
│   ├── proto/                      # Protocole CPE
│   └── crypto/                     # TinyCrypt (AES-128)
├── build/                          # Fichiers de build (généré)
├── out/                            # Firmware compilé (généré)
├── Makefile                        # Cibles de compilation
└── README.md                       # README projet principal
```

---

## 🔗 Liens externes utiles

| Ressource | Lien | Utilité |
|-----------|------|---------|
| **BBC Micro:bit** | https://microbit.org | Documentation officielle |
| **Yotta build** | http://yottabuild.org | Outil de build micro:bit |
| **TinyCrypt** | https://github.com/intel/tinycrypt | Bibliothèque crypto |
| **ARM Toolchain** | https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm | Compilateur |
| **NMEA GPS** | https://www.gpsinformation.org/dale/nmea.htm | Protocole GPS |

---

## ❓ FAQ

### Quelle est la portée maximale du système ?
**Réponse** : 100-150 m en extérieur dégagé, 30-50 m en intérieur. Voir [SYSTEME_COMPLET.md - Performances](SYSTEME_COMPLET.md#-performances-du-système).

### Combien de véhicules peuvent être gérés simultanément ?
**Réponse** : Théoriquement illimité en réception, ~15 msg/s en émission. Voir [SYSTEME_COMPLET.md - Débit](SYSTEME_COMPLET.md#débit).

### Comment changer la clé de chiffrement ?
**Réponse** : Générer avec `openssl rand -hex 16` puis éditer `CPE_KEY` dans tous les firmwares. Voir [PROTOCOLE_CPE.md - Gestion clé](PROTOCOLE_CPE.md#gestion-de-la-clé).

### Le système est-il certifié pour usage professionnel ?
**Réponse** : Non, c'est un prototype. Audit de sécurité requis pour déploiement opérationnel. Voir [SYSTEME_COMPLET.md - Avertissements](SYSTEME_COMPLET.md#️-avertissements).

### Quelle version de micro:bit est supportée ?
**Réponse** : Principalement v1 (nRF51822). v2 compatible mais non optimisé. Voir [APP_TERRAIN.md - Configuration](APP_TERRAIN.md#️-configuration-technique).

### Peut-on utiliser un autre module GPS que NEO-6M ?
**Réponse** : Oui, tout module NMEA compatible (UART). Adapter le parsing si besoin. Voir [APP_TERRAIN.md - Intégration GPS](APP_TERRAIN.md#-intégration-gps).

---

## 📝 Contribuer à la documentation

### Améliorer un document existant

1. Fork du projet
2. Éditer le fichier `.md` concerné
3. Respecter le format Markdown
4. Pull request avec description claire

### Ajouter un nouveau document

1. Créer fichier dans `docs/`
2. Ajouter lien dans ce README
3. Suivre le template des docs existantes :
   - Titre principal (#)
   - Vue d'ensemble
   - Sections numérotées
   - Exemples de code
   - Tableaux de référence
   - Liens vers autres docs

### Style guide

- ✅ Titres clairs et hiérarchisés
- ✅ Code formaté avec ```cpp ou ```bash
- ✅ Tableaux pour données structurées
- ✅ Emojis pour repères visuels (📡 📊 🔒 ⚠️)
- ✅ Liens relatifs entre documents
- ❌ Pas de terme technique sans explication
- ❌ Pas de code sans commentaire

---

## 📅 Historique des versions

| Version | Date | Changements |
|---------|------|-------------|
| 1.0 | Janvier 2026 | Première version complète |

---

## 📞 Support

**Projet** : IoT Terrain Micro:bit  
**Équipe** : Projet Transversal 2026  
**Licence** : Voir [LICENSE](../LICENSE)

---

**📚 Bonne lecture et bon développement !**
