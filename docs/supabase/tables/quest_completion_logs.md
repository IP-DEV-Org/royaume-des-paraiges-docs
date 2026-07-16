# Table: quest_completion_logs

Historique detaille de toutes les completions de quetes avec recompenses. Depuis la migration 066 (quêtes répétables), **une ligne par itération** : les montants loggés (`bonus_xp_awarded`, `bonus_cashback_awarded`, `coupon_template_id`, `target_value`) sont les valeurs **résolues** de l'itération (override `quest_iterations` ou fallback quête de base).

## Informations

| Propriete | Valeur |
|-----------|--------|
| **Schema** | `public` |
| **RLS** | Active |
| **Lignes** | -1 |

## Colonnes

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | `bigint` | Non | - | - |
| `quest_id` | `bigint` | Non | - | - |
| `quest_progress_id` | `bigint` | Non | - | - |
| `customer_id` | `uuid` | Non | - | - |
| `period_type` | `character varying(20)` | Non | - | - |
| `period_identifier` | `character varying(20)` | Non | - | - |
| `coupon_id` | `bigint` | Oui | - | - |
| `coupon_template_id` | `bigint` | Oui | - | - |
| `badge_awarded_id` | `bigint` | Oui | - | - |
| `bonus_xp_awarded` | `integer` | Non | 0 | - |
| `bonus_cashback_awarded` | `integer` | Non | 0 | - |
| `final_value` | `integer` | Non | - | `current_value` au moment de la distribution |
| `target_value` | `integer` | Non | - | Objectif **de l'itération** (override ou base) — migration 066 |
| `iteration` | `integer` | Non | 1 | **Migration 066.** Numéro d'itération (1 = première complétion de la période). UNIQUE avec `quest_progress_id`. |
| `completed_at` | `timestamp with time zone` | Non | now() | - |

## Cles primaires

- `id`

## Index uniques

- `quest_completion_logs_progress_iteration_key` UNIQUE (`quest_progress_id`, `iteration`) — invariant : une seule distribution par itération (migration 066).

## Relations (Foreign Keys)

- `quest_completion_logs_badge_awarded_id_fkey`: badge_awarded_id → user_badges.id
- `quest_completion_logs_coupon_id_fkey`: coupon_id → coupons.id
- `quest_completion_logs_coupon_template_id_fkey`: coupon_template_id → coupon_templates.id
- `quest_completion_logs_customer_id_fkey`: customer_id → profiles.id
- `quest_completion_logs_quest_id_fkey`: quest_id → quests.id
- `quest_completion_logs_quest_progress_id_fkey`: quest_progress_id → quest_progress.id
