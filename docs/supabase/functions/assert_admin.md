# Function: assert_admin

Helper d'autorisation introduit par la migration **040 (Security Definer hardening, 18/05/2026)**. Lève une exception `SQLSTATE 42501` si l'appelant n'est pas un admin (ou un bypass autorisé). Utilisé comme première instruction dans toutes les RPC `SECURITY DEFINER` réservées aux admins.

## Signature

```sql
CREATE FUNCTION public.assert_admin() RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
```

## Parametres

Aucun.

## Retour

`void`. Soit la fonction retourne silencieusement (autorisé), soit elle lève une exception.

## Logique

1. **Bypass automatique** (sans toucher à `profiles`) :
   - JWT claim `role = 'service_role'` (cas des Edge Functions et clients backend avec `service_role` key)
   - OU `session_user IN ('postgres', 'supabase_admin')` (cas des jobs `pg_cron`, des restaurations `pg_restore`, des appels directs `psql`)
2. Sinon, vérifie que `auth.uid() IS NOT NULL` (utilisateur authentifié). Sinon `RAISE EXCEPTION 'Non autorisé : authentification requise' USING ERRCODE = '42501'`.
3. Lit `profiles.role` pour `auth.uid()`.
4. Si `role IS DISTINCT FROM 'admin'` → `RAISE EXCEPTION 'Non autorisé : rôle admin requis' USING ERRCODE = '42501'`.

## Securite

- Marqué `SECURITY DEFINER` lui-même : la lecture de `profiles` se fait avec les droits du owner (postgres), donc bypass la RLS de `profiles` qui interdirait à l'appelant de lire d'autres lignes que la sienne. Ici on ne lit que sa propre ligne mais on évite le coût d'évaluation des policies.
- `REVOKE EXECUTE FROM PUBLIC, anon, authenticated` puis `GRANT EXECUTE TO authenticated`. `anon` ne peut donc pas appeler ce helper (et n'en a pas besoin, n'étant jamais admin).

### Piège — `session_user` vs `current_user`

À l'intérieur d'une fonction `SECURITY DEFINER`, `current_user` vaut toujours le owner de la fonction (`postgres`). Utiliser `current_user IN ('postgres', ...)` pour le bypass est donc **toujours vrai** et désactiverait complètement le guard. La migration 040 utilise donc **`session_user`** qui reflète l'identité réelle de la connexion (par exemple `authenticator` côté PostgREST, `postgres` pour pg_cron / `psql` direct).

## Utilisée par (migration 040)

- `credit_bonus_cashback`
- `create_manual_coupon`
- `distribute_quest_reward`
- `distribute_period_rewards_v2`
- `get_period_preview`
- `get_coupon_stats`
- `get_user_quests`
- `award_season_rank_badges`, `reset_season`, `snapshot_season`, `preview_season_closure`

## Voir aussi

- [`assert_admin_or_establishment_for`](./assert_admin_or_establishment_for.md) — autorise l'admin OU le gérant/employé de l'établissement ciblé
- [`assert_self_or_staff`](./assert_self_or_staff.md) — autorise l'utilisateur sur ses propres données OU n'importe quel staff
