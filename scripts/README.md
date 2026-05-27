# Scripts Utilitaires

Ce dossier contient les scripts utilitaires pour la gestion de la documentation et les migrations.

## Table des matieres

- [sync-supabase-docs.mjs](#sync-supabase-docsmjs) - Synchronisation de la documentation

---

## sync-supabase-docs.mjs

Script de synchronisation automatique de la documentation Supabase.

### Description

Ce script :
1. Se connecte a l'API Supabase Management
2. Recupere la configuration actuelle (tables, fonctions, triggers, vues, etc.)
3. Compare avec la documentation locale existante
4. Met a jour les fichiers de documentation si des differences sont detectees

### Prerequis

- Node.js 18+ (pour le support ESM natif)
- Un token d'acces Supabase Management API

### Configuration

Definir les variables d'environnement suivantes :

```bash
# Obligatoire - Token d'acces API Management Supabase
# Generer sur: https://app.supabase.com/account/tokens
export SUPABASE_ACCESS_TOKEN="sbp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

# Optionnel - ID du projet (defaut: kioysoveqemzjolfwpnu)
export SUPABASE_PROJECT_ID="kioysoveqemzjolfwpnu"
```

### Utilisation

```bash
# Depuis le dossier docs/
cd docs

# Mode dry-run (affiche les differences sans modifier)
node scripts/sync-supabase-docs.mjs --dry-run

# Mode dry-run avec details
node scripts/sync-supabase-docs.mjs --dry-run --verbose

# Appliquer les mises a jour
node scripts/sync-supabase-docs.mjs

# Avec verbose
node scripts/sync-supabase-docs.mjs --verbose
```

### Options

| Option | Description |
|--------|-------------|
| `--dry-run` | Affiche les differences sans modifier les fichiers |
| `--verbose` | Affiche des informations detaillees de debug |
| `--from-snapshot FILE` | Utilise un fichier JSON snapshot au lieu de l'API |
| `--save-snapshot FILE` | Sauvegarde les donnees dans un fichier JSON |

### Mode Snapshot (sans token API)

Un fichier `supabase-snapshot.json` est fourni avec les donnees actuelles du projet.
Cela permet de synchroniser la documentation sans avoir besoin d'un token API.

```bash
# Utiliser le snapshot fourni
node scripts/sync-supabase-docs.mjs --from-snapshot scripts/supabase-snapshot.json

# Avec dry-run
node scripts/sync-supabase-docs.mjs --from-snapshot scripts/supabase-snapshot.json --dry-run
```

Pour mettre a jour le snapshot (necessite un token API):

```bash
# Generer un nouveau snapshot
SUPABASE_ACCESS_TOKEN=xxx node scripts/sync-supabase-docs.mjs --save-snapshot scripts/supabase-snapshot.json
```

Le fichier snapshot peut aussi etre genere par Claude via les outils MCP Supabase.

### Fichiers generes/mis a jour

Le script gere les fichiers suivants :

```
docs/supabase/
├── configuration.md        # Configuration generale et resume
├── tables/
│   └── [table_name].md    # Un fichier par table
├── functions/
│   └── README.md          # Liste des fonctions PostgreSQL
├── views/
│   └── README.md          # Vues et vues materialisees
├── triggers/
│   └── README.md          # Liste des triggers
└── edge-functions/
    └── README.md          # Edge Functions deployes
```

### Exemple de sortie

```
========================================
  Synchronisation Documentation Supabase
========================================

Recuperation des donnees depuis Supabase...

Donnees recuperees:
  - 18 tables
  - 27 fonctions
  - 1 triggers
  - 4 vues materialisees
  - 1 vues
  - 2 enums
  - 3 jobs pg_cron
  - 1 edge functions
  - 6 extensions installees

Comparaison et mise a jour de la documentation...

[UPDATE] configuration.md
  → Fichier mis a jour: .../docs/supabase/configuration.md
[NEW] tables/new_table.md
  → Fichier cree: .../docs/supabase/tables/new_table.md
[OK] functions/README.md - Aucun changement

========================================
  Resume
========================================

  Fichiers crees:      1
  Fichiers mis a jour: 1
  Fichiers inchanges:  5

Termine!
```

### Integration CI/CD

Vous pouvez integrer ce script dans votre pipeline CI/CD pour verifier que la documentation est a jour :

```yaml
# .github/workflows/check-docs.yml
name: Check Supabase Docs

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 0 * * 1'  # Chaque lundi

jobs:
  check-docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Check for documentation drift
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
        run: |
          cd docs
          node scripts/sync-supabase-docs.mjs --dry-run
```

### Depannage

**Erreur: SUPABASE_ACCESS_TOKEN non defini**
- Verifiez que la variable d'environnement est correctement definie
- Generez un nouveau token sur https://app.supabase.com/account/tokens

**Erreur de connexion API**
- Verifiez votre connexion internet
- Verifiez que le token n'a pas expire
- Verifiez que le projet ID est correct

**Fichiers non mis a jour**
- Le script compare le contenu normalise (sans espaces/dates)
- Utilisez `--verbose` pour voir les details de comparaison
