# Function: assert_admin_or_establishment_for

Helper d'autorisation introduit par la migration **040 (Security Definer hardening, 18/05/2026)**. Autorise un admin **ou** un membre (`establishment` / `employee`) rattaché à un établissement précis. Utilisé par les RPC d'analytics scopées par etab.

## Signature

```sql
CREATE FUNCTION public.assert_admin_or_establishment_for(p_establishment_id bigint)
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
```

## Parametres

| Parametre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `p_establishment_id` | `BIGINT` | Oui | ID de l'établissement ciblé. Peut être `NULL` (autorisé uniquement pour admin / service_role / superuser). |

## Retour

`void`. Soit la fonction retourne silencieusement (autorisé), soit elle lève une exception `SQLSTATE 42501`.

## Logique

1. **Bypass automatique** (identique à [`assert_admin`](./assert_admin.md)) :
   - JWT claim `role = 'service_role'`
   - OU `session_user IN ('postgres', 'supabase_admin')`
2. Sinon, vérifie que `auth.uid() IS NOT NULL`. Sinon → `42501` « authentification requise ».
3. Lit `role` et `attached_establishment_id` de l'appelant depuis `profiles`.
4. Si `role = 'admin'` → autorisé (retour silencieux).
5. Sinon, si `role IN ('establishment', 'employee')` ET `attached_establishment_id IS NOT NULL` ET `attached_establishment_id = p_establishment_id` → autorisé.
6. Sinon → `RAISE EXCEPTION 'Non autorisé : accès limité à votre établissement' USING ERRCODE = '42501'`.

## Securite

- Marqué `SECURITY DEFINER` : bypass RLS sur `profiles` pour la lecture du rôle et de l'etab rattaché.
- `REVOKE EXECUTE FROM PUBLIC, anon, authenticated` puis `GRANT EXECUTE TO authenticated`.
- **Pattern anti-spoof obligatoire dans les callers** : avant d'appeler ce helper, les RPC analytics (`get_analytics_revenue` / `_debts` / `_stock`) **override** silencieusement `p_establishment_id := v_caller_etab` quand `role IN ('establishment', 'employee')`. Le client ne peut donc pas lire les analytics d'un autre etab en bidouillant le paramètre.

## Utilisée par (migration 040)

- `get_analytics_revenue`
- `get_analytics_debts`
- `get_analytics_stock`

## Voir aussi

- [`assert_admin`](./assert_admin.md) — admin strict
- [`assert_self_or_staff`](./assert_self_or_staff.md) — self ou staff
