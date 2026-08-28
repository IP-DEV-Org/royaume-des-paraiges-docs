# Table: reward_distribution_runs

Journal des exécutions de [`distribute_period_rewards_v2`](../functions/distribute_period_rewards_v2.md). Introduite par la migration **083 (28/08/2026)**.

## Pourquoi elle existe

Une distribution de classement qui ne récompense personne était jusqu'ici **indiscernable d'une distribution réussie** :

- les sorties anticipées de la RPC (« No reward tiers configured », « Rewards already distributed ») retournent un JSONB et n'écrivent nulle part ;
- un parcours sur un classement vide n'écrit rien non plus : ni `coupon_distribution_logs`, ni erreur ;
- `period_reward_configs` n'enregistre qu'un statut par période, écrit en fin de parcours : il dit « distribué », jamais « distribué à personne » ;
- pg_cron journalise `succeeded`, puisque le `SELECT` s'est bien exécuté.

C'est ce trou d'observabilité qui a laissé **onze distributions vides** passer inaperçues de janvier à avril 2026 (cf. migration 081).

**Une ligne par tentative**, pas par période — contrairement à `period_closures` (journal de la v1 legacy `distribute_leaderboard_rewards`, contraint à l'unicité sur la période, donc incapable de garder l'historique des tentatives successives).

## Colonnes

| Colonne | Type | Description |
|---------|------|-------------|
| `id` | `BIGSERIAL` | PK |
| `period_type` | `VARCHAR` | `weekly` / `monthly` / `yearly`. Volontairement **sans CHECK** : un type invalide doit pouvoir être journalisé. |
| `period_identifier` | `VARCHAR` | Période effectivement visée. `NULL` si le type était invalide. |
| `status` | `TEXT` | `success` / `partial` / `error` / `skipped`. Même vocabulaire que `email_report_runs`, donc même `<StatusBadge>` côté admin. |
| `reason` | `TEXT` | Détail machine : `no_active_tiers`, `already_distributed`, `empty_leaderboard`, `no_matching_tier`, `individual_failures`, `invalid_period_type`, `invalid_period_identifier`. |
| `origin` | `TEXT` | `cron` = aucun admin authentifié (pg_cron, service_role, SQL direct) ; `manual` = `auth.uid()` renseigné. |
| `rewards_distributed` | `INTEGER` | Nombre de récompenses effectivement accordées (coupon **ou** badge). |
| `leaderboard_size` | `INTEGER` | Joueurs classés dans la portée du barème (rangs 1..max(`rank_to`)). **C'est LA colonne qui aurait rendu le bug de la 081 visible** : elle valait 0 alors que la période close comptait 6 à 12 joueurs. |
| `errors` | `JSONB` | Erreurs individuelles rattrapées par le bloc EXCEPTION de la boucle. |
| `duration_ms` | `INTEGER` | Durée réelle (basée sur `EXTRACT(EPOCH ...)`, pas sur `MILLISECONDS` qui ignore les minutes). |
| `period_start` / `period_end` | `TIMESTAMPTZ` | Bornes issues de `get_period_bounds`. |
| `forced` | `BOOLEAN` | `p_force` de l'appel. |
| `triggered_by` | `UUID` | `auth.uid()`, FK `profiles` `ON DELETE SET NULL`. |
| `created_at` | `TIMESTAMPTZ` | |

## Sécurité

RLS activée. **Lecture** : admin uniquement (`profiles.role = 'admin'`). **Aucune policy d'écriture** : seule la RPC `SECURITY DEFINER`, propriétaire de la table, alimente le journal.

## À surveiller

Une ligne `status = 'skipped'` avec `reason = 'empty_leaderboard'` **alors que la période close comptait des joueurs** est le signal exact du bug de la 081. Une série de `no_active_tiers` signifie que le programme est à l'arrêt côté configuration - le moteur, lui, tourne.
