# Function: admin_has_any_feature

Migration **114 (09/09/2026)**. Variante multi-clés de `admin_has_feature` (migration 070) : vrai si l'appelant est super admin, ou admin disposant d'**au moins une** des fonctionnalités passées (aucune d'elles n'est dans [admin_disabled_features](../tables/admin_disabled_features.md)).

## Signature

```sql
CREATE FUNCTION public.admin_has_any_feature(VARIADIC p_feature_keys text[])
RETURNS boolean
LANGUAGE sql STABLE
SECURITY DEFINER
SET search_path = public
```

Appel : `admin_has_any_feature('achievements', 'rewards')` ou `admin_has_any_feature(VARIADIC ARRAY['achievements','rewards'])`.

## Pourquoi

Certaines tables sont écrites depuis deux pages du dashboard, donc deux clés de fonctionnalité : `badge_types` (lore des badges sur `/rewards/tiers`, badges succès sur `/rewards/achievements`), `coupons` (`/coupons/create` et `/rewards/cashback-gains/create`), les RPC RGPD (`/gdpr` et la zone dangereuse de `/users/[id]`). Un admin qui a l'une des deux pages doit pouvoir écrire.

## Utilisée par

Toutes les policies d'écriture posées par la migration 114 (générées par un helper temporaire `gate_writes`), [assert_admin_feature](./assert_admin_feature.md), et la branche « admin » de `gdpr_anonymize_user` / `gdpr_export_user_data`. Voir la section *Feature-gating* du [README des policies](../policies/README.md).

## Sécurité

`SECURITY DEFINER`, `REVOKE ALL FROM PUBLIC, anon`, `GRANT EXECUTE TO authenticated, service_role`. Jamais d'exception : conçue pour `USING` / `WITH CHECK`.
