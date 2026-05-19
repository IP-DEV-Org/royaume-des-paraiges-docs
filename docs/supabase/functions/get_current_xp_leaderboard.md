# Function: get_current_xp_leaderboard

Retourne le top N du leaderboard XP courant (semaine / mois / année en cours), avec pagination et compteur total.

Wrapper sécurisé des 3 vues matérialisées `weekly_xp_leaderboard` / `monthly_xp_leaderboard` / `yearly_xp_leaderboard` qui ne sont plus exposées directement via PostgREST depuis la migration `security_revoke_xp_leaderboard_matviews` (18/05/2026). Cette RPC n'expose volontairement pas la colonne `*_total_spent` (en €) pour respecter la règle produit « zéro euro côté client ».

## Signature

```sql
CREATE FUNCTION public.get_current_xp_leaderboard(
  p_period_type TEXT,
  p_limit INTEGER DEFAULT 50,
  p_offset INTEGER DEFAULT 0
) RETURNS TABLE (
  customer_id UUID,
  total_xp BIGINT,
  receipt_count BIGINT,
  establishment_count BIGINT,
  rank BIGINT,
  total_count BIGINT
)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public, pg_temp
```

## Paramètres

| Paramètre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `p_period_type` | `TEXT` | Oui | - | `'weekly'`, `'monthly'` ou `'yearly'`. Toute autre valeur lève `SQLSTATE 22023`. |
| `p_limit` | `INTEGER` | Non | `50` | Taille de page |
| `p_offset` | `INTEGER` | Non | `0` | Offset de pagination |

## Retour

Une ligne par utilisateur classé dans la période courante :

| Colonne | Description |
|---------|-------------|
| `customer_id` | FK `profiles.id` |
| `total_xp` | XP gagnés sur la période (alias de `weekly_xp` / `monthly_xp` / `yearly_xp`) |
| `receipt_count` | Nombre de tickets sur la période |
| `establishment_count` | Nombre d'établissements distincts visités sur la période |
| `rank` | Rang dans le classement de la période |
| `total_count` | Total d'utilisateurs classés (répété sur chaque ligne pour permettre la pagination côté client) |

## Autorisations

- `GRANT EXECUTE TO anon, authenticated` — leaderboard public, lisible sans login.

## Consommateurs

- `royaume-paraiges-front/src/features/gains/services/leaderboardService.ts` — méthodes `getWeeklyLeaderboard`, `getMonthlyLeaderboard`, `getYearlyLeaderboard` (regroupées dans le helper privé `fetchCurrentLeaderboard`).

## Notes

- **Pas de `*_total_spent`** : la colonne est délibérément exclue du `RETURNS TABLE`. Elle reste dans les MV sous-jacentes pour des usages admin futurs mais ne traverse jamais PostgREST côté client.
- **Bigint** : PostgREST sérialise les `bigint` en string. Les consommateurs doivent appliquer `Number()`.
- **Pattern jumeau** : voir [`get_current_xp_rank`](./get_current_xp_rank.md) pour le rang d'un utilisateur précis sans charger tout le leaderboard.
- **Filtre `NOT p.is_test`** (migration 040, 18/05/2026) : la RPC joint désormais `profiles` et filtre les comptes de test à la fois pour les lignes retournées **et** pour le `total_count`. Cohérent avec `get_period_leaderboard` et `get_public_xp_leaderboard`. Les MV sous-jacentes ne filtrent pas — le filtre est appliqué au niveau RPC pour éviter d'avoir à les régénérer.
- **Colonnes** : `total_xp` (alias des colonnes `weekly_xp` / `monthly_xp` / `yearly_xp` selon la période), **pas** `xp`. `total_count` (compteur global d'utilisateurs classés), **pas** `total`.
