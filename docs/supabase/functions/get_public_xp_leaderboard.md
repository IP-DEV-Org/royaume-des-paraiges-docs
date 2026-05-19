# Function: get_public_xp_leaderboard

Retourne le top N du leaderboard XP **all-time** (tous temps confondus, agrège l'XP cumulée depuis l'inscription).

Wrapper sécurisé de la MV `user_stats`, fermée à l'API depuis la migration `security_revoke_user_stats_matview` (18/05/2026) car elle exposait des montants en euros (`cashback_*`) et d'autres champs non publics.

## Signature

```sql
CREATE FUNCTION public.get_public_xp_leaderboard(p_limit INTEGER)
RETURNS TABLE (customer_id UUID, total_xp BIGINT)
LANGUAGE sql
SECURITY DEFINER
SET search_path = public, pg_temp
```

## Paramètres

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `p_limit` | `INTEGER` | Oui | Taille de page (top N) |

## Retour

Une ligne par utilisateur classé all-time : `(customer_id, total_xp)`. Filtré : `is_test = false` (cf. migration 040 du 18/05/2026 qui ajoute le `JOIN profiles` + `WHERE NOT p.is_test`) et `total_xp IS NOT NULL`.

## Autorisations

- `GRANT EXECUTE TO anon, authenticated` — leaderboard public.

## Consommateurs

- `royaume-paraiges-front/src/features/gains/services/leaderboardService.ts` — méthode `getGlobalLeaderboard`.

## Notes

- **Cible**: leaderboard « tous temps » (différent des leaderboards périodiques `get_current_xp_leaderboard`).
- **Pas de cashback ni montants €** : la RPC n'expose que `customer_id` et `total_xp` ; les colonnes `cashback_earned`/`cashback_available`/`cashback_spent` de `user_stats` ne sont accessibles que via [`get_user_stats`](./create_receipt.md) (rôle admin/employee) ou [`get_unspent_cashback_total`](./get_unspent_cashback_total.md) (admin).
- **Migration introductrice** : `security_user_stats_rpc_wrappers` (18/05/2026).
