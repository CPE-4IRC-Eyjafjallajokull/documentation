# DAST (Dynamic Application Security Testing)

## Table des matières

1. [Introduction](#introduction)
2. [Intérêt du DAST](#intérêt-du-dast)
3. [Raisons du choix](#raisons-du-choix)
4. [Vue d’ensemble de l’architecture DAST](#vue-densemble-de-larchitecture-dast)
5. [Implémentation par application](#implémentation-par-application)
	1. [app-qg-api (API FastAPI)](#1-app-qg-api-api-fastapi)
	2. [app-qg-front (Front Next.js)](#2-app-qg-front-front-nextjs)
6. [Configuration ZAP et règles](#configuration-zap-et-règles)
7. [Bonnes pratiques et stratégie de déclenchement](#bonnes-pratiques-et-stratégie-de-déclenchement)
8. [Limites du DAST et complémentarité avec le SAST](#limites-du-dast-et-complémentarité-avec-le-sast)

---

## Introduction

Le DAST (Dynamic Application Security Testing) est une approche de tests de sécurité qui consiste à analyser une application **en fonctionnement**, en interagissant avec ses endpoints HTTP comme le ferait un client réel (navigateur, API client, attaquant). Contrairement au SAST, qui inspecte le code source, le DAST observe le comportement de l’application déployée.

Dans ce projet transversal, le DAST est mis en œuvre avec **OWASP ZAP** exécuté depuis des workflows GitHub Actions dédiés, via le fichier `zap-scan.yml` pour :

- l’API backend `app-qg-api` ;
- le front `app-qg-front` (Next.js).

Ces scans sont intégrés dans la CI/CD et peuvent être lancés à la demande, sur chaque push/PR, ainsi que de façon planifiée (cron).

---

## Intérêt du DAST

### 1. Tester l’application « telle qu’elle tourne »

Le DAST permet de détecter des vulnérabilités **au niveau runtime** :

- Mauvaise configuration HTTP (en-têtes manquants, redirections, cookies) ;
- Failles d’injection (SQL, XSS, commande système) réellement exploitables ;
- Problèmes d’authentification, de session ou de contrôle d’accès visibles côté HTTP ;
- Exposition d’informations sensibles dans les réponses.

L’intérêt est de valider la sécurité **dans l’environnement le plus proche de la production** (Docker Compose, configuration Next.js, variables d’environnement, proxies, etc.).

### 2. Complémentarité avec le SAST

Le SAST identifie les faiblesses dans le code, mais certaines vulnérabilités ne se révèlent qu’en exécution :

- Mauvais câblage d’un middleware de sécurité ;
- Erreur de configuration d’un reverse proxy ou d’un header ;
- Enchaînement de redirections ;
- Erreurs d’implémentation dans l’infrastructure.

Le DAST permet d’attraper ces cas en simulant le comportement d’un attaquant externe.

### 3. Vérification des protections de surface

OWASP ZAP vérifie notamment :

- La présence et la configuration de **CSP**, **X-Frame-Options**, **X-Content-Type-Options** ;
- La gestion des cookies (Secure, HttpOnly, SameSite) ;
- Les redirections HTTP ;
- La présence ou absence de protections spécifiques à certaines vulnérabilités.

### 4. Automatisation dans la CI/CD

En intégrant ZAP dans GitHub Actions :

- Les scans sont exécutés automatiquement à chaque **push** et **pull request** ;
- Des scans plus complets sont programmés via **cron** ;
- Les rapports sont archivés en artefacts et peuvent créer des **issues GitHub** en cas d’alertes critiques.

---

## Raisons du choix

### Choix d’OWASP ZAP

OWASP ZAP (Zed Attack Proxy) est un proxy de test d’intrusion applicatif open-source maintenu par la fondation OWASP. Les raisons principales de son choix :

1. **Outil de référence OWASP**
	- Aligné avec les recommandations **OWASP Top 10** ;
	- Reconnu dans l’industrie, largement documenté et maintenu.

2. **Intégration GitHub Actions**
	- Actions officielles : `zaproxy/action-api-scan`, `zaproxy/action-baseline`, `zaproxy/action-full-scan` ;
	- Configuration simple via YAML, avec support natif de l’export de rapports.

3. **Modes de scans variés**
	- **API Scan** pour les APIs REST/JSON (cas de `app-qg-api`) ;
	- **Baseline Scan** pour un crawl non-intrusif (cas de `app-qg-front`) ;
	- **Full Scan** pour un scan plus agressif, réservé aux exécutions planifiées.

4. **Personnalisation fine des règles**
	- Fichiers `.zap/rules.tsv` pour adapter les règles : IGNORE / WARN / FAIL ;
	- Possibilité de réduire les faux positifs propres à FastAPI/Next.js.

5. **Coût**
	- Outil open-source, sans coût de licence ;
	- Utilisable facilement dans un contexte pédagogique et de projet étudiant.

### Choix de l’intégration dans GitHub Actions

- Centralisation de la sécurité (SAST, DAST, scans de dépendances) dans le même outil CI ;
- Historique complet des exécutions et rapports dans GitHub ;
- Triggers sur `push`, `pull_request`, `schedule` et `workflow_dispatch` pour couvrir :
  - les changements de code,
  - les revues de code,
  - et les scans réguliers indépendants des commits.

---

## Vue d’ensemble de l’architecture DAST

Schéma logique simplifié :

```mermaid
graph LR
	 A[Développeur push/PR] --> B[GitHub Actions]
	 B --> C[Démarrage de l'application cible]
	 C --> D[OWASP ZAP]
	 D --> E[Rapports DAST]
	 E --> F[Artefacts GitHub]
	 E --> G[Issues GitHub (si échec)]
```

Étapes typiques d’un workflow ZAP :

1. Checkout du dépôt ;
2. Détection de la présence d’un `docker-compose.yml` ;
3. Démarrage de l’application :
	- soit via `docker compose up`,
	- soit via l’écosystème natif (uv pour Python, npm pour Next.js) ;
4. Vérification de la disponibilité de la cible (healthcheck) ;
5. Lancement du scan ZAP (API, baseline ou full) ;
6. Traitement des résultats :
	- échec en cas d’alertes High/Critical,
	- upload des rapports,
	- création optionnelle d’issues GitHub.

---

## Implémentation par application

### 1. app-qg-api (API FastAPI)

Workflow : `.github/workflows/zap-scan.yml` dans le répertoire `app-qg-api`.

#### Déclencheurs

```yaml
on:
  push:
	 branches: [main]

  pull_request:
	 branches: [main]

  schedule:
	 - cron: "0 3 * * 2"  # Tous les mardis à 3h UTC

  workflow_dispatch:
```

Ce workflow couvre :

- les commits sur la branche principale ;
- les PR vers `main` ;
- un scan hebdomadaire planifié ;
- un déclenchement manuel à la demande.

#### Démarrage de l’API cible

Le job `zap-scan` commence par détecter la présence d’un `docker-compose.yml` :

- Si **docker-compose présent** :
  - Génération d’un fichier `.env` minimal pour l’API (DSN PostgreSQL, DSN RabbitMQ, variables applicatives) ;
  - Démarrage complet de la stack via `docker compose up -d` ;
  - Attente fixe pour laisser le temps de démarrer.

- Si **docker-compose absent** :
  - Installation de Python 3.12 via `actions/setup-python` ;
  - Installation de `uv` et des dépendances (`uv sync`) ;
  - Démarrage de l’API FastAPI via `uv run uvicorn app.main:app --host 0.0.0.0 --port 8000` en arrière-plan, avec stockage du PID dans `.api.pid` ;
  - Attente du démarrage.

Un healthcheck est ensuite effectué sur `http://localhost:8000/health` :

```python
@router.get("/health", tags=["meta"])
async def healthcheck() -> dict[str, str]:
	 return {"status": "ok", "version": settings.app.version}
```

Le workflow attend que cet endpoint réponde avec succès avant de lancer ZAP.

#### Scans OWASP ZAP

Deux types de scans sont configurés :

1. **API Scan** pour les `push` et `pull_request` :

	```yaml
	uses: zaproxy/action-api-scan@v0.9.0
	with:
	  target: "http://localhost:8000"
	  rules_file_name: ".zap/rules.tsv"
	  cmd_options: "-a -j -l WARN"
	  issue_title: "ZAP API Scan - Résultats de sécurité"
	  fail_action: false
	```

	- `target` : URL racine de l’API ;
	- `rules_file_name` : configuration spécifique FastAPI (voir plus bas) ;
	- `cmd_options` :
	  - `-a` : mode agressif,
	  - `-j` : sortie JSON,
	  - `-l WARN` : niveau de journalisation ;
	- `fail_action: false` : le job ne s’arrête pas immédiatement sur la base des alertes, la logique métier de fail est gérée dans un step dédié.

2. **Full Scan** pour les exécutions `schedule` et `workflow_dispatch` :

	```yaml
	uses: zaproxy/action-full-scan@v0.10.0
	with:
	  target: "http://localhost:8000"
	  rules_file_name: ".zap/rules.tsv"
	  cmd_options: "-a -j -l WARN"
	```

	Le full scan est plus intrusif (tests actifs plus nombreux) et est donc réservé aux scans hors PR pour éviter des temps d’exécution trop longs.

#### Politique d’échec en fonction des alertes

Après l’exécution de ZAP, un step spécifique lit `report_json.json` à l’aide de `jq` et compte les alertes de sévérité **HIGH** :

```bash
high_count=$(jq '[.site[]?.alerts[]? | select((.risk // "") | ascii_upcase == "HIGH")] | length' report_json.json)
if [ "$high_count" -gt 0 ]; then
  exit 1
fi
```

- Si au moins une alerte de sévérité HIGH est détectée, le job échoue ;
- Les détails des alertes sont affichés dans les logs GitHub Actions.

#### Rapports et artefacts

Les rapports ZAP (`report_html.html`, `report_json.json`, `report_md.md`) sont :

- sauvegardés en tant qu’artefacts GitHub pendant 30 jours ;
- utilisables pour l’analyse manuelle ou la documentation.

En cas d’échec du workflow (présence d’alertes graves), une issue GitHub est créée automatiquement, incluant le rapport Markdown.

#### Configuration des règles ZAP (FastAPI)

Fichier : `.zap/rules.tsv` dans `app-qg-api`.

Exemples de règles :

- **IGNORE** : règles non pertinentes ou générant des faux positifs dans ce contexte (par exemple absence d’anti-CSRF pour des APIs JWT) ;
- **WARN** : problèmes à surveiller mais non bloquants (CSP scanner, headers manquants) ;
- **FAIL** : failles critiques (XSS, injection SQL, code injection, XXE, etc.).

---

### 2. app-qg-front (Front Next.js)

Workflow : `.github/workflows/zap-scan.yml` dans le répertoire `app-qg-front`.

#### Déclencheurs

Identiques à ceux de `app-qg-api` : `push`, `pull_request`, `schedule` (mardi 3h), `workflow_dispatch`.

#### Démarrage de l’application cible

Le workflow gère deux cas, comme pour l’API :

- **Avec `docker-compose.yml`** :
  - Génération d’un `.env` avec les variables essentielles (API_URL, KEYCLOAK_ISSUER, secrets de test, etc.) ;
  - `docker compose up -d --build` ;
  - attente du démarrage.

- **Sans `docker-compose.yml`** :
  - Installation de Node.js 20 avec cache npm ;
  - `npm ci` pour une installation déterministe ;
  - `npm run build` en mode production ;
  - `npm run start` sur le port 3000, PID stocké dans `.nextjs.pid` ;
  - attente du démarrage.

Le workflow vérifie ensuite que `http://localhost:3000` répond correctement avant de lancer ZAP.

#### Scans OWASP ZAP

Deux modes sont configurés :

1. **Baseline Scan** pour `push`/`pull_request` :

	```yaml
	uses: zaproxy/action-baseline@v0.15.0
	with:
	  target: "http://localhost:3000"
	  rules_file_name: ".zap/rules.tsv"
	  cmd_options: "-a -j -l WARN"
	```

	Le baseline scan réalise un crawl passif et quelques tests légers, adapté aux PR pour obtenir un feedback rapide sans impacter fortement le temps de CI.

2. **Full Scan** pour `schedule` et `workflow_dispatch` :

	```yaml
	uses: zaproxy/action-full-scan@v0.10.0
	with:
	  target: "http://localhost:3000"
	  rules_file_name: ".zap/rules.tsv"
	  cmd_options: "-a -j -l WARN"
	```

	Le full scan va plus loin : tests actifs plus nombreux, plus grande couverture fonctionnelle.

#### Politique d’échec

Une logique similaire à celle de l’API est utilisée :

- Lecture de `report_json.json` ;
- Comptage des alertes de sévérité HIGH ;
- Échec du job si au moins une alerte critique est détectée.

#### Rapports

Les rapports ZAP sont également sauvegardés en artefacts pendant 30 jours.

#### Configuration des règles ZAP (Next.js)

Fichier : `.zap/rules.tsv` dans `app-qg-front`.

Principes :

- Certaines règles sont **ignorées** car Next.js gère déjà certains en-têtes ou patterns modernes ;
- Les problèmes de configuration (headers, politique de contenu, etc.) sont en **WARN** ;
- Les failles graves (XSS, injection, code injection) sont en **FAIL**.

---


## Configuration ZAP et règles

Les fichiers `.zap/rules.tsv` permettent de contrôler finement le comportement de ZAP.

Format :

```text
rule_id<TAB>ACTION<TAB>description
```

Où `ACTION` peut être :

- `IGNORE` : la règle est ignorée ;
- `WARN` : la règle est exécutée mais ne fait pas échouer le scan ;
- `FAIL` : la règle fait échouer le scan si une alerte est déclenchée.

### Exemple FastAPI (app-qg-api)

- Ignorer certains faux positifs :
  - Absence d’anti-CSRF (API JWT) ;
  - Détails de timestamp.
- Prévenir sur la configuration des headers de sécurité (CSP, X-Frame-Options, etc.) ;
- Faire échouer sur :
  - injections SQL (MySQL/PostgreSQL/SQLite) ;
  - XSS (reflected, stored, etc.) ;
  - injections de code serveur (OS command, code injection) ;
  - XXE.

### Exemple Next.js (app-qg-front)

- Ignorer certains patterns propres aux applis modernes ;
- Mettre en WARN les problèmes de CSP / en-têtes de sécurité ;
- Faire échouer sur les vulnérabilités d’injection critiques (XSS, code injection, etc.).

Cette configuration limite les faux positifs tout en gardant une exigence forte sur les vulnérabilités graves.

---

## Bonnes pratiques et stratégie de déclenchement

### Stratégie de déclenchement

Les workflows ZAP suivent une stratégie cohérente :

- `push` sur `main` : vérifier que le code mergé ne casse pas la sécurité dynamique ;
- `pull_request` vers `main` : offrir un feedback aux reviewers avant le merge ;
- `schedule` : scans hebdomadaires plus lourds, indépendants des commits ;
- `workflow_dispatch` : possibilité de lancer un scan ponctuel lors d’une investigation.

### Gestion des permissions

Les jobs disposent des permissions GitHub minimales nécessaires :

- lecture du code ;
- écriture d’événements de sécurité ;
- création d’issues si besoin.

### Gestion des ressources

- `timeout-minutes: 30` pour éviter des workflows bloqués ;
- Démarrage et arrêt explicites des services (Docker ou processus locaux) ;
- Attente explicite du démarrage via healthcheck/HTTP 200.

### Politique d’échec

- Le workflow n’échoue pas directement sur la base de la sortie de l’action ZAP (`fail_action: false`) ;
- Une étape dédiée lit les rapports JSON et impose la politique :
  - échec si des alertes de sévérité HIGH (ou plus) sont présentes ;
  - possibilité d’ajuster plus finement si besoin (par exemple en tolérant certains contextes sur un environnement de test).

---

## Limites du DAST et complémentarité avec le SAST

### Limites du DAST

- Ne voit que la surface exposée (endpoints HTTP accessibles) ;
- Peut manquer des chemins d’exécution internes non atteignables ;
- Peut générer des faux positifs ou perturber des environnements fragiles ;
- Temps d’exécution parfois long selon la taille de l’application.

### Complémentarité avec le SAST

Le SAST (CodeQL, linters, etc.) et le DAST (ZAP) sont complémentaires :

- SAST : trouve des failles potentielles dans le code, même si elles ne sont pas encore exposées ;
- DAST : vérifie que les protections sont effectivement en place et efficaces en runtime ;
- Ensemble : ils réduisent fortement la probabilité de laisser passer une vulnérabilité exploitable en production.

---

## Conclusion

Le DAST mis en place dans ce projet, basé sur OWASP ZAP et intégré à GitHub Actions, permet :

- de valider la sécurité des applications **en conditions réelles d’exécution** ;
- de détecter des vulnérabilités non visibles par les seuls outils SAST ;
- d’obtenir des rapports détaillés et historisés ;
- d’automatiser les réactions (échec du pipeline, création d’issues).

Ce dispositif complète la stratégie SAST et les scans de dépendances pour offrir une approche de sécurité applicative globale, reproductible et adaptée à un contexte multi-langages et multi-applications.



