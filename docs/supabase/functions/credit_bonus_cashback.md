# Function: credit_bonus_cashback

Credite un bonus cashback directement au solde d'un utilisateur via la table `gains`.

## Signature

```sql
CREATE FUNCTION credit_bonus_cashback(
  p_customer_id UUID,
  p_amount INTEGER,
  p_coupon_id BIGINT DEFAULT NULL,
  p_source_type VARCHAR DEFAULT 'bonus_cashback_manual',
  p_period_identifier VARCHAR DEFAULT NULL
) RETURNS BIGINT
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public'
```

## Parametres

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `p_customer_id` | `UUID` | Oui | - | ID du client (profiles.id) |
| `p_amount` | `INTEGER` | Oui | - | Montant en centimes |
| `p_coupon_id` | `BIGINT` | Non | `NULL` | ID du coupon source (tracabilite) |
| `p_source_type` | `VARCHAR` | Non | `'bonus_cashback_manual'` | Type de source |
| `p_period_identifier` | `VARCHAR` | Non | `NULL` | Identifiant de periode (ex: 2026-W07, 2026-02) |

## Valeurs de p_source_type

| Valeur | Utilise par |
|--------|-------------|
| `bonus_cashback_manual` | `create_manual_coupon()` |
| `bonus_cashback_leaderboard` | `distribute_period_rewards_v2()` |
| `bonus_cashback_quest` | `distribute_quest_reward()` |
| `bonus_cashback_trigger` | `create_weekly_coupon()` |
| `bonus_cashback_migration` | Migration des anciens coupons |

## Retour

Retourne l'`id` du gain cree (`BIGINT`).

## Actions

1. `PERFORM public.assert_admin()` — bloque tout appelant non-admin (voir Securite)
2. Valide `p_amount > 0` et `p_customer_id IS NOT NULL`
3. Insere un `gains` avec `receipt_id=NULL`, `establishment_id=NULL`, `xp=0`, `cashback_money=p_amount`, `period_identifier=p_period_identifier`
4. Rafraichit la vue materialisee `user_stats` (CONCURRENTLY)

## Securite

Depuis la migration **040 (Security Definer hardening, 18/05/2026)** :

- Première instruction : `PERFORM public.assert_admin()`. Voir [`assert_admin.md`](./assert_admin.md).
- **REVOKE EXECUTE FROM PUBLIC, anon, authenticated** — aucun GRANT n'est ré-accordé. La fonction n'est donc **plus appelable directement via PostgREST** (REST/RPC).
- Reste appelable :
  - en interne via la chaîne `SECURITY DEFINER` (le owner = `postgres` conserve l'EXECUTE) depuis [`create_manual_coupon`](./create_manual_coupon.md), [`distribute_period_rewards_v2`](./distribute_period_rewards_v2.md), [`distribute_quest_reward`](./distribute_quest_reward.md), le trigger `distribute_quest_rewards` et `create_weekly_coupon` ;
  - via `service_role` (Edge Functions, cron) ou superuser.
- L'admin n'a plus besoin d'appeler `credit_bonus_cashback` directement — il passe toujours par `create_manual_coupon` qui crée le coupon source et délègue le crédit en interne.

## Exemple

```sql
SELECT credit_bonus_cashback(
  '123e4567-e89b-12d3-a456-426614174000'::UUID,
  500,  -- 5 EUR
  42,   -- coupon_id
  'bonus_cashback_manual',
  '2026-W07'  -- period_identifier (optionnel)
);
```

## Notes

- Appelee automatiquement par les fonctions de distribution quand un coupon a un montant fixe
- Le coupon associe doit etre cree au prealable avec `used=true`
