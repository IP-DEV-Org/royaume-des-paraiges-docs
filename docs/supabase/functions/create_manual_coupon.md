# Function: create_manual_coupon

Cree un coupon manuellement pour un client. Si le coupon est a montant fixe, il est automatiquement converti en bonus cashback (credite immediatement au solde du client).

## Signature

```sql
CREATE FUNCTION create_manual_coupon(
  p_customer_id UUID,
  p_template_id BIGINT DEFAULT NULL,
  p_amount INTEGER DEFAULT NULL,
  p_percentage INTEGER DEFAULT NULL,
  p_expires_at TIMESTAMPTZ DEFAULT NULL,
  p_validity_days INTEGER DEFAULT NULL,
  p_notes TEXT DEFAULT NULL
) RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public'
```

## Parametres

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `p_customer_id` | `UUID` | Oui | - | ID du client (profiles.id) |
| `p_template_id` | `BIGINT` | Non | `NULL` | ID du template a utiliser |
| `p_amount` | `INTEGER` | Non | `NULL` | Montant en centimes (si pas de template) |
| `p_percentage` | `INTEGER` | Non | `NULL` | Pourcentage (si pas de template) |
| `p_expires_at` | `TIMESTAMPTZ` | Non | `NULL` | Date d'expiration explicite |
| `p_validity_days` | `INTEGER` | Non | `NULL` | Jours de validite (alternative a expires_at) |
| `p_notes` | `TEXT` | Non | `NULL` | Notes admin (logguees dans distribution_logs) |

> **Migration 040 (18/05/2026)** : le paramètre `p_admin_id` a été **retiré**. L'identité de l'admin pour l'audit trail (`coupon_distribution_logs.distributed_by`) est désormais dérivée de `auth.uid()` côté serveur — elle n'est plus falsifiable depuis le client.

## Retour

Retourne un objet `JSONB` :

```json
{
  "success": true,
  "coupon_id": 123,
  "customer_id": "uuid",
  "amount": 500,
  "percentage": null,
  "expires_at": null,
  "is_bonus_cashback": true,
  "gain_id": 456
}
```

## Logique

1. `PERFORM public.assert_admin()` — bloque les appelants non-admin (voir Securite)
2. Verifie que le customer existe
3. Si `p_template_id` fourni, charge le template et utilise ses valeurs (amount/percentage/validity_days)
4. Sinon, utilise `p_amount` ou `p_percentage` (l'un ou l'autre, pas les deux)
5. Determine si c'est un bonus cashback : `amount IS NOT NULL AND percentage IS NULL`
6. Cree le coupon :
   - **Bonus cashback** : `used=true`, `expires_at=NULL`
   - **Pourcentage** : `used=false`, avec expiration calculee
7. Si bonus cashback, appelle `credit_bonus_cashback()` avec `source_type='bonus_cashback_manual'`
8. Log dans `coupon_distribution_logs` avec `distribution_type='manual'` et `distributed_by = auth.uid()`

## Securite

Depuis la migration **040 (Security Definer hardening, 18/05/2026)** :

- Première instruction du body : `PERFORM public.assert_admin()`. Lève `SQLSTATE 42501` si l'appelant n'est pas admin (ou `service_role` / superuser pour le bypass automatisé). Voir [`assert_admin.md`](./assert_admin.md).
- L'audit trail (`coupon_distribution_logs.distributed_by`) est dérivé de `auth.uid()` directement dans la fonction — **plus de paramètre falsifiable** `p_admin_id`.
- `GRANT EXECUTE TO authenticated` (filtré par le guard), `REVOKE EXECUTE FROM PUBLIC, anon`.

## Exemple

```sql
-- Via template
SELECT create_manual_coupon(
  '123e4567-e89b-12d3-a456-426614174000'::UUID,
  p_template_id := 5,
  p_notes := 'Geste commercial'
);

-- Via montant fixe (bonus cashback)
SELECT create_manual_coupon(
  '123e4567-e89b-12d3-a456-426614174000'::UUID,
  p_amount := 500,  -- 5 EUR
  p_notes := 'Compensation'
);

-- Via pourcentage
SELECT create_manual_coupon(
  '123e4567-e89b-12d3-a456-426614174000'::UUID,
  p_percentage := 10,
  p_validity_days := 30,
  p_notes := 'Reduction 10%'
);
```

## Notes

- Appelee depuis l'admin via `couponService.ts`
- Les coupons montant fixe sont immediatement convertis en bonus cashback
- Les coupons pourcentage restent des coupons classiques utilisables sur commande
