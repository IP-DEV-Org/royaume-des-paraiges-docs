# Tables - Royaume des Paraiges

## Vue d'ensemble

La base de données contient **33 tables** dans le schéma `public`. Toutes les tables ont **RLS activé**.

## Diagramme des Relations

```
┌─────────────────┐
│   auth.users    │
└────────┬────────┘
         │ 1:1
         ▼
┌─────────────────┐     ┌─────────────────┐
│    profiles     │◄────│     likes       │
│                 │◄────│    comments     │
│                 │◄────│     notes       │
│                 │◄────│    coupons      │
│                 │◄────│   user_badges   │
│                 │◄────│   spendings     │
│                 │◄────│ quest_progress  │
└────────┬────────┘     └─────────────────┘
         │ 1:N
         ▼
┌─────────────────┐     ┌─────────────────┐
│    receipts     │────►│     gains       │
│                 │────►│  receipt_lines  │
│                 │────►│   spendings     │
└─────────────────┘     └─────────────────┘

┌─────────────────┐     ┌─────────────────┐
│   badge_types   │────►│   user_badges   │
│                 │────►│  reward_tiers   │
│                 │────►│     quests      │
└─────────────────┘     └─────────────────┘

┌─────────────────────────────────────────┐
│   Système de Coupons Administrable      │
├─────────────────────────────────────────┤
│   coupon_templates ──► reward_tiers     │
│          │                   │          │
│          ▼                   ▼          │
│       coupons    coupon_distribution_logs│
│                                         │
│   period_reward_configs                 │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│   Système de Quêtes (défis / missions)  │
├─────────────────────────────────────────┤
│   quests ──► quest_progress             │
│      │              │                   │
│      ▼              ▼                   │
│   quest_periods   quest_completion_logs │
│                                         │
│   available_periods                     │
└─────────────────────────────────────────┘

> **Terminologie produit** : les tables `quests` / `quest_progress` couvrent les **défis** (quêtes récurrentes, weekly/monthly/yearly — seul modèle implémenté aujourd'hui). Les **missions** (quêtes ponctuelles, one-shot) sont un modèle différé. Le mot « quête » employé seul est ambigu — préciser « récurrente » ou « ponctuelle ». Voir `tables/quests.md` pour le détail.

┌─────────────────────────────────────────┐
│   Contenu (tables de contenu)             │
├─────────────────────────────────────────┤
│   breweries ──► beers                   │
│                    │                    │
│   beer_styles ─────┼── beers_beer_styles│
│                    │                    │
│   establishments ──┼── beers_establishments│
│        │           │                    │
│        └───────────┼── news_establishments│
│                    │                    │
│   news ────────────┘                    │
│                                         │
│   level_thresholds                      │
└─────────────────────────────────────────┘
```

## Liste des Tables

### Tables Principales

| Table | Description | FK vers |
|-------|-------------|---------|
| [profiles](./profiles.md) | Profils utilisateurs | auth.users |
| [receipts](./receipts.md) | Tickets de caisse | profiles |
| [receipt_lines](./receipt_lines.md) | Lignes de paiement | receipts |
| [gains](./gains.md) | XP et cashback gagnés | receipts |

### Tables de Dépenses et Récompenses

| Table | Description | FK vers |
|-------|-------------|---------|
| [spendings](./spendings.md) | Dépenses de cashback | profiles, receipts |
| [coupons](./coupons.md) | Coupons de réduction | profiles, coupon_templates |

### Tables d'Interaction Sociale

| Table | Description | FK vers |
|-------|-------------|---------|
| [likes](./likes.md) | Likes sur contenus | profiles |
| [comments](./comments.md) | Commentaires | profiles |
| [notes](./notes.md) | Notes/évaluations | profiles |

### Tables de Badges et Leaderboard

| Table | Description | FK vers |
|-------|-------------|---------|
| [badge_types](./badge_types.md) | Définitions des badges | - |
| [user_badges](./user_badges.md) | Badges obtenus | profiles, badge_types |
| [leaderboard_reward_distributions](./leaderboard_reward_distributions.md) | Historique récompenses | profiles, coupons |
| [period_closures](./period_closures.md) | Clôtures de périodes | - |

### Tables du Système de Coupons Administrable

| Table | Description | FK vers |
|-------|-------------|---------|
| [coupon_templates](./coupon_templates.md) | Modèles de coupons réutilisables | profiles |
| [reward_tiers](./reward_tiers.md) | Paliers de récompenses leaderboard | coupon_templates, badge_types |
| [period_reward_configs](./period_reward_configs.md) | Config personnalisée par période | profiles |
| [coupon_distribution_logs](./coupon_distribution_logs.md) | Historique des distributions | profiles, coupons, coupon_templates, reward_tiers |

### Tables du Système de Quêtes

| Table | Description | FK vers |
|-------|-------------|---------|
| [quests](./quests.md) | Définition des quêtes périodiques | coupon_templates, badge_types, profiles |
| [quest_progress](./quest_progress.md) | Progression des utilisateurs | quests, profiles |
| [quest_completion_logs](./quest_completion_logs.md) | Historique des complétions | quests, quest_progress, profiles, coupons |
| [quest_periods](./quest_periods.md) | Liaison quêtes-périodes | quests |
| [available_periods](./available_periods.md) | Périodes disponibles | - |

### Tables de Contenu

| Table | Description | FK vers |
|-------|-------------|---------|
| [beers](./beers.md) | Catalogue des bières | breweries |
| [breweries](./breweries.md) | Brasseries | - |
| [beer_styles](./beer_styles.md) | Styles de bières | - |
| [establishments](./establishments.md) | Établissements partenaires | - |
| [news](./news.md) | Actualités | - |
| [level_thresholds](./level_thresholds.md) | Seuils de niveaux | - |

### Tables de Liaison (Many-to-Many)

| Table | Description | FK vers |
|-------|-------------|---------|
| [beers_establishments](./beers_establishments.md) | Bières-Établissements | beers, establishments |
| [beers_beer_styles](./beers_beer_styles.md) | Bières-Styles | beers, beer_styles |
| [news_establishments](./news_establishments.md) | News-Établissements | news, establishments |

### Tables de Configuration

| Table | Description |
|-------|-------------|
| [constants](./constants.md) | Constantes de configuration |
| [legal_pages](./legal_pages.md) | Pages legales (CGU, confidentialite) |

## Types Personnalisés (Enums)

### user_role

Rôles des utilisateurs dans l'application.

```sql
CREATE TYPE user_role AS ENUM (
  'client',       -- Utilisateur standard
  'employee',     -- Employé d'établissement
  'establishment', -- Compte établissement
  'admin'         -- Administrateur
);
```

### payment_method

Méthodes de paiement acceptées.

```sql
CREATE TYPE payment_method AS ENUM (
  'card',     -- Carte bancaire
  'cash',     -- Espèces
  'cashback', -- Utilisation du solde cashback
  'coupon'    -- Deprecated: les coupons sont geres via gains/bonus cashback depuis fev 2026
);
```

### quest_type

Types de quêtes disponibles.

```sql
CREATE TYPE quest_type AS ENUM (
  'xp_earned',              -- Gagner X XP
  'amount_spent',           -- Dépenser X centimes (saisie en € côté admin, ×100 au submit)
  'establishments_visited', -- Visiter X établissements
  'orders_count',           -- Passer X commandes
  'quest_completed',        -- Compléter N sous-périodes de quêtes
  'consumption_count',      -- Consommer X produits du type consumption_type
  'cashback_earned'         -- Collecter X Paraiges de Bronze (migration 028)
);
```

## Contraintes Uniques

| Table | Contrainte | Colonnes |
|-------|------------|----------|
| `profiles` | `profiles_username_key` | `username` |
| `badge_types` | `badge_types_slug_key` | `slug` |
| `quests` | `quests_slug_key` | `slug` |
| `user_badges` | `user_badges_customer_id_badge_id_period_identifier_key` | `customer_id`, `badge_id`, `period_identifier` |
| `leaderboard_reward_distributions` | unique | `customer_id`, `period_type`, `period_identifier` |
| `period_closures` | unique | `period_type`, `period_identifier` |
| `period_reward_configs` | unique | `period_type`, `period_identifier` |
| `quest_progress` | unique | `quest_id`, `customer_id`, `period_identifier` |
| `quest_periods` | unique | `quest_id`, `period_identifier` |
| `available_periods` | unique | `period_type`, `period_identifier` |

## Notes sur les Montants

**Important** : Tous les montants monétaires sont stockés en **centimes** (INTEGER).

- `amount = 100` → 1,00€
- `amount = 390` → 3,90€
- `amount = 5000` → 50,00€

Cela évite les problèmes de précision des nombres à virgule flottante.

## Dernière mise à jour

- **Date**: 2026-02-17
