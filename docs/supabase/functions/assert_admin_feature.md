# Function: assert_admin_feature

Migration **114 (09/09/2026)**. Pendant de [assert_admin](./assert_admin.md) pour les RPC `SECURITY DEFINER` réservées à **une page** du dashboard : même bypass, même errcode, plus la vérification que l'appelant dispose d'au moins une des fonctionnalités passées.

## Signature

```sql
CREATE FUNCTION public.assert_admin_feature(VARIADIC p_feature_keys text[])
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
```

## Logique

1. **Bypass** identique à `assert_admin` : JWT `role = 'service_role'`, ou `session_user IN ('postgres', 'supabase_admin')` (pg_cron, psql). Retour silencieux.
2. `PERFORM public.assert_admin()` : authentification + rôle admin, mêmes messages `42501` qu'avant.
3. Si `NOT admin_has_any_feature(VARIADIC p_feature_keys)` : `RAISE EXCEPTION 'FEATURE_DISABLED: fonctionnalité … désactivée pour ce compte' USING ERRCODE = '42501'`.

## Utilisée par (migration 114)

| RPC | Clés |
|---|---|
| `create_manual_coupon` | `coupons`, `cashback-gains` |
| `distribute_period_rewards_v2`, `snapshot_season`, `award_season_rank_badges`, `reset_season` | `rewards` |
| `distribute_quest_reward`, `distribute_all_quest_rewards` | `quests` |
| `admin_delete_receipt`, `admin_reset_identity_photo_cooldown` | `users` |

`credit_bonus_cashback` garde `assert_admin` : helper interne appelé par le trigger des quêtes et par la distribution, jamais par le dashboard directement. Les RPC de lecture (analytics, aperçus) ne sont pas gatées.

## Piège de test

Impossible de tester le refus en `SET ROLE authenticated` depuis une session `postgres` : `session_user` reste `postgres`, le bypass s'applique. Tester avec un vrai JWT (PostgREST) ou vérifier `admin_has_any_feature(...)` seule.

## Sécurité

`REVOKE ALL FROM PUBLIC, anon`, `GRANT EXECUTE TO authenticated, service_role`. La migration **115** a aussi retiré `anon` de `admin_reset_identity_photo_cooldown` (signalé par l'advisor Supabase).
