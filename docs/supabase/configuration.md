# Documentation Supabase - Royaume des Paraiges

## Informations Projet

| Propriete | Valeur |
|-----------|--------|
| **Project ID** | `kioysoveqemzjolfwpnu` |
| **Region** | eu-west-3 (Paris) |
| **URL API** | `https://kioysoveqemzjolfwpnu.supabase.co` |
| **PostgreSQL** | 17.6.1.003 |

## Vue d'ensemble

Cette documentation decrit la configuration complete de la base de donnees Supabase pour l'application Royaume des Paraiges.

## Structure de la Documentation

```
docs/docs/supabase/
├── configuration.md             # Ce fichier
├── README.md                    # Vue d'ensemble
├── tables/                      # Structure des tables
│   ├── README.md               # Vue d'ensemble des tables
│   └── [table_name].md         # Documentation par table
├── functions/                   # Fonctions PostgreSQL
│   └── README.md               # Index des fonctions
├── triggers/                    # Triggers
│   └── README.md
├── policies/                    # Politiques RLS
│   └── README.md
├── views/                       # Vues et vues materialisees
│   └── README.md
├── storage/                     # Buckets et policies storage
│   └── README.md
└── edge-functions/             # Edge Functions
    └── README.md
```

## Resume de la Base de Donnees

### Tables Public (30)

| Table | Lignes | RLS | Description |
|-------|--------|-----|-------------|
| `available_periods` | 130 | Oui | Periodes disponibles pour les quetes |
| `badge_types` | 9 | Oui | Definitions des badges disponibles dans le systeme |
| `beers` | 196 | Oui | Catalogue des bieres |
| `beers_beer_styles` | 285 | Oui | Liaison M2M bieres-styles |
| `beers_establishments` | 43 | Oui | Liaison M2M bieres-etablissements |
| `beer_styles` | 47 | Oui | Styles de bieres |
| `breweries` | 66 | Oui | Brasseries |
| `comments` | 2 | Oui | Commentaires utilisateurs |
| `constants` | 2 | Oui | Constantes systeme |
| `coupon_distribution_logs` | 0 | Oui | Historique complet de toutes les distributions de coupons |
| `coupon_templates` | 23 | Oui | Modeles de coupons reutilisables pour les recompenses |
| `coupons` | 2 | Oui | Coupons utilisateurs |
| `establishments` | 7 | Oui | Etablissements partenaires |
| `gains` | 58 | Oui | Gains XP et cashback |
| `leaderboard_reward_distributions` | 0 | Oui | Historique des recompenses distribuees aux gagnants du leaderboard |
| `level_thresholds` | 30 | Oui | Seuils de niveaux utilisateur |
| `likes` | 8 | Oui | Likes utilisateurs |
| `news` | 2 | Oui | Actualites |
| `news_establishments` | 3 | Oui | Liaison M2M news-etablissements |
| `notes` | 0 | Oui | Notes utilisateurs |
| `period_closures` | 1 | Oui | Tracking des periodes de leaderboard fermees et recompenses distribuees |
| `period_reward_configs` | 0 | Oui | Configuration personnalisee des recompenses pour une periode specifique |
| `profiles` | 17 | Oui | Profils utilisateurs |
| `quest_completion_logs` | 2 | Oui | Historique detaille de toutes les completions de quetes avec recompenses |
| `quest_periods` | 5 | Oui | Table de liaison pour assigner des quetes a des periodes specifiques |
| `quest_progress` | 3 | Oui | Suivi de la progression des utilisateurs sur les quetes |
| `quests` | 3 | Oui | Definition des quetes periodiques disponibles |
| `receipt_lines` | 58 | Oui | Lignes de paiement des tickets |
| `receipts` | 58 | Oui | Tickets de caisse |
| `reward_tiers` | 6 | Oui | Paliers de recompenses pour le leaderboard (configurable par rang) |
| `spendings` | 5 | Oui | Depenses cashback |
| `user_badges` | 2 | Oui | Collection des badges obtenus par chaque utilisateur |

### Types Personnalises (Enums)

| Enum | Valeurs |
|------|---------|
| `payment_method` | card, cash, cashback, coupon |
| `quest_type` | xp_earned, amount_spent, cashback_earned, establishments_visited, orders_count, quest_completed, consumption_count |
| `user_role` | client, employee, establishment, admin |

### Fonctions PostgreSQL (39)

| Fonction | Arguments | Retour | Description |
|----------|-----------|--------|-------------|
| `award_user_badge` | p_customer_id, p_badge_slug, p_period_type, p_period_identifier, p_rank | json | Attribue un badge a un utilisateur pour une periode donnee |
| `calculate_gains` | p_amount_for_gains | jsonb | Calcule les gains XP/cashback |
| `calculate_quest_progress` | p_customer_id, p_quest_id, p_period_identifier | integer | Calcule la progression actuelle d'un utilisateur pour une quete donnee |
| `check_cashback_balance` | p_customer_id, p_cashback_requested | jsonb | Verifie le solde cashback |
| `check_email_exists` | email_to_check | boolean | Verifie si un email existe |
| `check_period_closed` | p_period_type, p_period_identifier | boolean | Verifie si une periode de leaderboard a deja ete fermee |
| `create_frequency_coupon` | p_customer_id | json | Cree un coupon de 5% pour 10+ commandes/semaine |
| `create_leaderboard_reward_coupon` | p_customer_id, p_amount, p_percentage | json | Cree un coupon de recompense leaderboard |
| `create_manual_coupon` | p_customer_id, p_template_id, p_amount, p_percentage, ... | jsonb | Cree un coupon manuellement |
| `create_receipt` | p_customer_id, p_establishment_id, p_payment_methods, p_coupon_ids | jsonb | Cree un recu avec paiements, coupons et gains |
| `create_spending_from_cashback_payment` | - | trigger | Trigger: cree spending sur paiement cashback |
| `create_weekly_coupon` | p_customer_id | json | Cree un coupon 3,90€ pour 50€+ depenses/semaine |
| `distribute_all_quest_rewards` | p_admin_id | json | Distribue les recompenses pour toutes les quetes completees |
| `distribute_leaderboard_rewards` | p_period_type, p_force | json | **Legacy, ne pas appeler** : barèmes en dur, lit les vues materialisees et ignore la periode (defaut corrige sur la v2 par la migration 081). Aucun appelant. |
| `distribute_period_rewards_v2` | p_period_type, p_period_identifier, p_preview_only, p_force | jsonb | Distribue les recompenses leaderboard avec tiers configurables. Admin only (migration 040, audit trail via `auth.uid()`). Depuis la migration 081 : classement lu via `get_period_leaderboard` (et non plus les vues materialisees), periode par defaut = periode **close**. |
| `distribute_quest_reward` | p_quest_progress_id | json | Distribue les recompenses pour une quete completee. Admin only (migration 040, audit trail via `auth.uid()`). |
| `distribute_quest_rewards` | - | trigger | Trigger: distribue recompenses quand quete completee |
| `expire_quest_progress` | - | json | Expire les quest_progress dont la periode est terminee |
| `get_coupon_stats` | - | jsonb | Retourne les statistiques globales des coupons |
| `get_current_user_role` | - | text | Retourne le role de l'utilisateur courant |
| `get_customer_available_coupons` | p_customer_id | TABLE | Recupere les coupons disponibles d'un client |
| `get_period_bounds` | p_period_type, p_period_identifier | TABLE | Retourne les bornes d'une periode |
| `get_period_identifier` | p_period_type, p_date | varchar | Calcule l'identifiant de periode (2026-W04, 2026-01, 2026) |
| `get_period_preview` | p_period_type, p_period_identifier | jsonb | Previsualise la distribution des recompenses |
| `get_user_badges` | p_customer_id | TABLE | Recupere tous les badges d'un utilisateur |
| `get_user_cashback_balance` | p_customer_id | jsonb | Retourne le solde cashback d'un utilisateur |
| `get_user_complete_stats` | p_customer_id | jsonb | Retourne toutes les statistiques d'un utilisateur |
| `get_user_info` | user_ids | TABLE | Recupere les infos de plusieurs utilisateurs |
| `get_user_quests` | p_customer_id, p_period_type | json | Recupere les quetes actives avec progression |
| `get_user_xp_stats` | p_customer_id | jsonb | Retourne les XP et statistiques d'un utilisateur |
| `handle_new_user` | - | trigger | Trigger: cree profil automatiquement |
| `handle_user_delete` | - | trigger | Trigger: supprime profil automatiquement |
| `sync_auth_to_profiles` | - | TABLE | Synchronise auth.users vers public.profiles |
| `trigger_update_quest_progress` | - | trigger | Trigger: met a jour progression quetes |
| `update_profile_from_auth` | user_id | void | Met a jour un profil depuis auth.users |
| `update_quest_progress_for_receipt` | p_receipt_id | json | Met a jour la progression de toutes les quetes apres un receipt |
| `validate_coupons` | p_customer_id, p_coupon_ids | jsonb | Valide une liste de coupons |
| `validate_payment_methods` | p_payment_methods | jsonb | Valide les methodes de paiement |

### Triggers (3)

| Trigger | Table | Fonction | Description |
|---------|-------|----------|-------------|
| `trigger_distribute_quest_rewards` | `quest_progress` | `distribute_quest_rewards` | Distribue les recompenses quand une quete est completee |
| `trigger_create_spending_on_cashback` | `receipt_lines` | `create_spending_from_cashback_payment` | Cree un spending lors d'un paiement cashback |
| `trigger_quest_progress_on_receipt` | `receipts` | `trigger_update_quest_progress` | Met a jour la progression des quetes apres un receipt |

### Jobs pg_cron (4)

| Job | Schedule | Commande | Description |
|-----|----------|----------|-------------|
| 1 | `5 0 * * 1` | `SELECT distribute_period_rewards_v2('weekly')` | Lundi 00:05 - Distribution de la **semaine close** (migration 081 : sans identifiant, la RPC vise la periode precedente, pas celle qui vient de s'ouvrir) |
| 2 | `10 0 1 * *` | `SELECT distribute_period_rewards_v2('monthly')` | 1er du mois 00:10 - Distribution du **mois clos** |
| 3 | `15 0 1 1 *` | `SELECT distribute_period_rewards_v2('yearly')` | 1er janvier 00:15 - Distribution de l'**annee close** |
| 6 | `0 0 * * *` | `SELECT expire_quest_progress()` | Tous les jours a minuit - Expiration quetes non completees |

### Vues Materialisees (4)

| Vue | Description |
|-----|-------------|
| `weekly_xp_leaderboard` | Leaderboard hebdomadaire (lundi-dimanche). ⚠️ Periode **cablee sur `now()`** au moment du `REFRESH` : utilisable pour le classement temps reel (`get_current_xp_leaderboard`, `get_current_xp_rank`, `create_receipt`), jamais pour une periode close - utiliser `get_period_leaderboard`. |
| `monthly_xp_leaderboard` | Leaderboard mensuel (1er au dernier jour du mois) |
| `yearly_xp_leaderboard` | Leaderboard annuel (1er janvier au 31 decembre) |
| `user_stats` | Vue materialisee combinant les statistiques XP et cashback par utilisateur |

### Vues (1)

| Vue | Description |
|-----|-------------|
| `reward_distribution_stats` | Statistiques de distribution des recompenses |

### Storage Buckets (1)

| Bucket | Description |
|--------|-------------|
| `avatars` | Photos de profil (public) |

### Edge Functions (1)

| Fonction | Description |
|----------|-------------|
| `send-contact-email` | Envoi d'email de contact (JWT requis) |

## Extensions Installees

| Extension | Schema | Version | Description |
|-----------|--------|---------|-------------|
| `pg_cron` | pg_catalog | 1.6.4 | Job scheduler for PostgreSQL |
| `pg_graphql` | graphql | 1.5.11 | GraphQL support |
| `pg_stat_statements` | extensions | 1.11 | Track SQL statements statistics |
| `pgcrypto` | extensions | 1.3 | Cryptographic functions |
| `plpgsql` | pg_catalog | 1.0 | PL/pgSQL procedural language |
| `supabase_vault` | vault | 0.3.1 | Supabase Vault Extension |
| `uuid-ossp` | extensions | 1.1 | Generate UUIDs |

## Schema Relationnel

```
auth.users
    │
    └──► profiles (id = auth.uid())
            │
            ├──► likes (user_id)
            ├──► comments (customer_id)
            ├──► notes (customer_id)
            ├──► coupons (customer_id)
            ├──► user_badges (customer_id)
            ├──► leaderboard_reward_distributions (customer_id)
            ├──► spendings (customer_id)
            ├──► coupon_distribution_logs (customer_id, distributed_by)
            ├──► coupon_templates (created_by)
            ├──► period_reward_configs (distributed_by)
            ├──► quest_progress (customer_id)
            ├──► quest_completion_logs (customer_id)
            │
            └──► receipts (customer_id)
                    │
                    ├──► receipt_lines (receipt_id)
                    ├──► gains (receipt_id)
                    └──► spendings (receipt_id)

badge_types ──► user_badges (badge_id)
            └──► reward_tiers (badge_type_id)
            └──► quests (badge_type_id)

coupons ◄── leaderboard_reward_distributions (coupon_amount_id, coupon_percentage_id)
        ◄── coupon_distribution_logs (coupon_id)
        ◄── quest_completion_logs (coupon_id)

coupon_templates ──► coupons (template_id)
                 └──► reward_tiers (coupon_template_id)
                 └──► coupon_distribution_logs (coupon_template_id)
                 └──► quests (coupon_template_id)
                 └──► quest_completion_logs (coupon_template_id)

reward_tiers ──► coupon_distribution_logs (tier_id)

quests ──► quest_progress (quest_id)
       └──► quest_completion_logs (quest_id)
       └──► quest_periods (quest_id)

breweries ──► beers (brewery_id)

beers ──► beers_establishments (beer_id)
      └──► beers_beer_styles (beer_id)

beer_styles ──► beers_beer_styles (beer_style_id)

establishments ──► beers_establishments (establishment_id)
               └──► news_establishments (establishment_id)

news ──► news_establishments (news_id)
```

## Migrations

Total: **72 migrations** appliquees

Dernieres migrations:
- `20260209140738` - update_distribute_period_rewards_v2_with_period_v2
- `20260209140636` - update_distribute_quest_reward_with_period
- `20260209140616` - update_distribute_quest_rewards_trigger_with_period
- `20260209140555` - update_credit_bonus_cashback_with_period
- `20260209140542` - add_period_identifier_to_gains
- `20260209133926` - fix_gains_rls_policy_for_bonus_gains
- `20260209130009` - drop_duplicate_calculate_quest_progress
- `20260209110824` - create_legal_pages_table
- `20260208232416` - fix_check_cashback_balance_include_all_gains
- `20260208230901` - fix_distribute_quest_rewards_missing_customer_id

## Derniere mise a jour

- **Date** : 2026-02-17
- **Generee par** : Claude Code
