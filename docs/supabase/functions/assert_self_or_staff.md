# Function: assert_self_or_staff

Helper d'autorisation introduit par la migration **040 (Security Definer hardening, 18/05/2026)**. Autorise un utilisateur à lire / agir sur ses propres données **OU** n'importe quel staff (admin / employee / establishment). Utilisé par les RPC où un client doit pouvoir consulter sa progression personnelle pendant qu'un staff peut consulter celle de n'importe qui.

## Signature

```sql
CREATE FUNCTION public.assert_self_or_staff(p_customer_id uuid)
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
```

## Parametres

| Parametre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `p_customer_id` | `UUID` | Oui | ID du `profiles` ciblé par l'opération. |

## Retour

`void`. Soit la fonction retourne silencieusement (autorisé), soit elle lève une exception `SQLSTATE 42501`.

## Logique

1. **Bypass automatique** (identique à [`assert_admin`](./assert_admin.md)) :
   - JWT claim `role = 'service_role'`
   - OU `session_user IN ('postgres', 'supabase_admin')`
2. Sinon, vérifie que `auth.uid() IS NOT NULL`. Sinon → `42501` « authentification requise ».
3. Si `auth.uid() = p_customer_id` → autorisé (l'utilisateur agit sur ses propres données).
4. Sinon, lit `profiles.role` de l'appelant.
5. Si `role IN ('admin', 'employee', 'establishment')` → autorisé (n'importe quel staff peut agir sur n'importe quel client).
6. Sinon → `RAISE EXCEPTION 'Non autorisé : self, employee, establishment ou admin requis' USING ERRCODE = '42501'`.

## Securite

- Marqué `SECURITY DEFINER` : bypass RLS sur `profiles` pour la lecture du rôle.
- `REVOKE EXECUTE FROM PUBLIC, anon, authenticated` puis `GRANT EXECUTE TO authenticated`.
- À utiliser dans toute RPC qui agit pour-le-compte-d'un-customer-id donné en paramètre.

## Utilisée par (migration 040)

- `calculate_quest_progress` — un client lit sa propre progression, un staff (employé en salle, gérant, admin) peut lire celle de n'importe quel client.

## Voir aussi

- [`assert_admin`](./assert_admin.md) — admin strict
- [`assert_admin_or_establishment_for`](./assert_admin_or_establishment_for.md) — admin ou rattaché à un etab précis
