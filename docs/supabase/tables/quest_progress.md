# Table: quest_progress

Suivi de la progression des utilisateurs sur les **défis** (quêtes récurrentes). Remis à zéro à chaque nouvelle période (`period_identifier`). Les **missions** (quêtes ponctuelles, one-shot) ne passent pas par cette table — modèle différé, voir `tables/quests.md` → « Terminologie produit ».

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
| `customer_id` | `uuid` | Non | - | - |
| `period_type` | `character varying(20)` | Non | - | - |
| `period_identifier` | `character varying(20)` | Non | - | - |
| `current_value` | `integer` | Non | 0 | Cumul brut de la période (non borné au target) |
| `target_value` | `integer` | Non | - | Copie du target de base de la quête (itération 1) |
| `status` | `character varying(20)` | Non | 'in_progress'::character varying | Voir « Machine à états (migration 066) » ci-dessous |
| `completions_count` | `integer` | Non | 0 | **Migration 066.** Nombre d'itérations déjà récompensées sur la période. **Monotone** : jamais décrémenté, même si `current_value` rebaisse (ex. `admin_delete_receipt`) — on ne reprend jamais une récompense distribuée. Porte l'idempotence du trigger de distribution. |
| `completed_at` | `timestamp with time zone` | Oui | - | - |
| `rewarded_at` | `timestamp with time zone` | Oui | - | - |
| `created_at` | `timestamp with time zone` | Non | now() | - |
| `updated_at` | `timestamp with time zone` | Non | now() | - |

## Cles primaires

- `id`

## Relations (Foreign Keys)

- `quest_progress_customer_id_fkey`: customer_id → profiles.id
- `quest_progress_quest_id_fkey`: quest_id → quests.id

## Machine à états (migration 066 — quêtes répétables)

Depuis la migration 066, le trigger `distribute_quest_rewards` (BEFORE INSERT OR UPDATE via touch) est **l'autorité finale sur `status`** :

- `in_progress` — itérations restantes possibles (y compris après 1+ complétions déjà récompensées).
- `completed` — **transitoire** : posé par l'upsert de `update_quest_progress_for_receipt` ou par `update_meta_quest_progress`, immédiatement résolu par le trigger en `rewarded` ou `in_progress`. Ne persiste plus en base (les 30 lignes `completed` historiques pré-066 sont des reliquats du bug « complétion au premier INSERT jamais récompensée », corrigé par la 066).
- `rewarded` — toutes les itérations **autorisées** ont été payées (`completions_count >= plafond`). Peut **ré-ouvrir** en `in_progress` si le plafond monte en cours de période (level-up du joueur ou changement de barème).
- `expired` — période terminée. Ligne définitivement gelée : le trigger refuse toute distribution sur une ligne expirée.

L'itération k est débloquée quand `current_value >= somme des objectifs des itérations 1..k` (objectif par itération = override `quest_iterations` ou `quests.target_value`). Une seule vague peut payer plusieurs itérations d'un coup.
