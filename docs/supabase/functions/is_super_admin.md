# Function: is_super_admin

Helper d'autorisation introduit par la migration **057 (juillet 2026)** avec le système d'accès par fonctionnalité entre admins. Retourne `true` si le profil visé est un super admin (`profiles.is_super_admin = true` **et** `role = 'admin'`), `false` sinon (y compris profil inexistant).

## Signature

```sql
CREATE FUNCTION public.is_super_admin(p_profile_id uuid DEFAULT auth.uid())
RETURNS boolean
LANGUAGE sql STABLE
SECURITY DEFINER
SET search_path = public
```

## Parametres

| Paramètre | Type | Default | Description |
|---|---|---|---|
| `p_profile_id` | uuid | `auth.uid()` | Profil à tester. Sans argument, teste l'appelant. |

## Retour

`boolean` — jamais NULL (`COALESCE(..., false)`).

## Logique

```sql
SELECT COALESCE(
  (SELECT p.is_super_admin AND p.role = 'admin' FROM public.profiles p WHERE p.id = p_profile_id),
  false);
```

Le double critère `is_super_admin AND role = 'admin'` garantit qu'une rétrogradation du rôle (admin → autre) suffit à retirer le statut effectif de super admin, même si le flag reste `true`.

## Securite

- `SECURITY DEFINER` : lit `profiles` avec les droits du owner (bypass RLS) — nécessaire car les policies de `admin_disabled_features` doivent pouvoir tester le statut d'un **autre** profil que l'appelant.
- `REVOKE EXECUTE FROM PUBLIC, anon` puis `GRANT EXECUTE TO authenticated, service_role`. Requis par les policies RLS de `admin_disabled_features`, évaluées avec les droits de l'appelant.
- Contrairement aux helpers `assert_*`, ne lève pas d'exception : conçue pour les clauses `USING` / `WITH CHECK` des policies.

## Utilisée par

- Policies RLS de [`admin_disabled_features`](../tables/admin_disabled_features.md) (`read own or super admin`, `super admin insert`, `super admin delete`).
- Trigger `trg_protect_role` sur `profiles` (migrations 059-060) : seul un super admin peut changer le `role` d'un profil.
- UI admin : sélecteur de rôle de l'onglet Édition (`/users/[id]`) désactivé pour les non-super-admins.

## Voir aussi

- [`assert_admin`](./assert_admin.md) — guard bloquant (exception 42501) pour les RPC admin
- Trigger `trg_protect_is_super_admin` sur `profiles` — rend `is_super_admin` immuable hors `service_role`/`postgres` (anti-escalade ; cf. [`profiles`](../tables/profiles.md))
