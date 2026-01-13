# Gitleaks - Détection de secrets

## Introduction

**Gitleaks** est un outil open-source qui détecte les secrets (clés API, tokens, mots de passe, clés privées) exposés dans le code source et l'historique Git.

Il est déployé via **GitHub Actions** sur toutes nos applications :
- `app-qg-api`
- `app-qg-front`
- `app-qg-java-engine`
- `app-terrain-front`

---

## Pourquoi Gitleaks ?

| Critère | Description |
|---------|-------------|
| **Prévention** | Détecte les secrets avant qu'ils n'atteignent la branche principale |
| **Historique Git** | Analyse tous les commits, pas seulement le code actuel |
| **Performance** | Écrit en Go, très rapide même sur de gros dépôts |
| **Intégration** | Rapport SARIF compatible avec GitHub Security |
| **Gratuit** | Open-source, licence MIT |

---

## Workflow GitHub Actions

Fichier : `.github/workflows/gitleaks.yml` (identique dans chaque application)

```yaml
name: Gitleaks Secret Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 4 * * 1"  # Tous les lundis à 4h UTC
  workflow_dispatch:

jobs:
  gitleaks:
    name: Gitleaks Secret Scan
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install Gitleaks
        run: |
          wget -q https://github.com/gitleaks/gitleaks/releases/download/v8.21.2/gitleaks_8.21.2_linux_x64.tar.gz
          tar -xzf gitleaks_8.21.2_linux_x64.tar.gz
          sudo mv gitleaks /usr/local/bin/
          gitleaks version

      - name: Run Gitleaks scan
        run: gitleaks detect --source . --verbose --report-format sarif --report-path gitleaks.sarif --exit-code 1

      - name: Upload SARIF results
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: gitleaks.sarif
          category: gitleaks
```

### Déclencheurs

| Événement | Quand |
|-----------|-------|
| `push` | Push sur `main` |
| `pull_request` | PR vers `main` |
| `schedule` | Lundi 4h UTC |
| `workflow_dispatch` | Manuel |

---

## Bonnes pratiques

### ✅ À faire

- Utiliser des **variables d'environnement** au lieu de hardcoder les secrets
- Ajouter `.env` dans `.gitignore`
- Utiliser **GitHub Secrets** pour la CI/CD
- Fournir un `.env.example` sans vraies valeurs

### ❌ À éviter

- Committer des fichiers `.env` avec des vraies valeurs
- Hardcoder des credentials dans le code

---

## En cas de secret détecté

1. **Révoquer** immédiatement le secret (régénérer la clé/token)
2. **Supprimer** de l'historique Git si nécessaire :
   ```bash
   git filter-repo --replace-text <(echo 'SECRET==>***REMOVED***')
   ```
3. **Mettre à jour** les services utilisant ce secret

---

## Ressources

- [Documentation Gitleaks](https://github.com/gitleaks/gitleaks)
- [GitHub Code Scanning](https://docs.github.com/en/code-security/code-scanning)