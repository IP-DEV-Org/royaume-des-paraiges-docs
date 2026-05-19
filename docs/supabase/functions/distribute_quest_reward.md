# Function: distribute_quest_reward

Distribue manuellement la récompense d'une quête complétée (coupon + badge + bonus XP/cashback) à partir d'une ligne de `quest_progress` au statut `completed`. Admin only.

Distinct du trigger automatique `distribute_quest_rewards()` (sans `_reward` au singulier) qui s'exécute à chaque INSERT/UPDATE de `quest_progress`. Ce wrapper RPC est appelé depuis l'admin pour forcer / rejouer une distribution.

## Signature

```sql
CREATE FUNCTION public.distribute_quest_reward(p_quest_progress_id BIGINT)
RETURNS json
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
```

## Parametres

| Parametre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `p_quest_progress_id` | `BIGINT` | Oui | ID de la ligne `quest_progress` à distribuer. Doit être au statut `completed` (sinon erreur). |

> **Migration 040 (18/05/2026)** : le paramètre `p_admin_id` a été **retiré**. L'audit trail (`coupon_distribution_logs.distributed_by` indirectement via `credit_bonus_cashback`, et `quest_completion_logs`) utilise `auth.uid()`.

## Retour

```json
{
  "success": true,
  "quest_id": 5,
  "quest_name": "Champion hebdo",
  "customer_id": "uuid",
  "period_identifier": "2026-W21",
  "is_bonus_cashback": true,
  "rewards": {
    "coupon_id": 123,
    "gain_id": 456,
    "badge_id": 12,
    "bonus_xp": 50,
    "bonus_cashback": 200
  }
}
```

Cas d'erreur (retourne `{ "success": false, "error": "..." }`) :
- `Quest progress not found`
- `Quest already rewarded` (statut = `rewarded`)
- `Quest not completed yet` (statut ≠ `completed`)

## Logique

1. `PERFORM public.assert_admin()` — bloque les non-admins (voir Securite).
2. Charge la ligne `quest_progress`. Si introuvable → erreur.
3. Vérifie statut = `completed` (sinon erreur dédiée).
4. Charge la `quest` associée.
5. Si `quest.coupon_template_id` défini :
   - Charge `coupon_templates`.
   - Determine `is_bonus_cashback = (amount IS NOT NULL AND percentage IS NULL)`.
   - INSERT `coupons` (`distribution_type = 'quest'`, `period_identifier`).
   - Si bonus cashback : appelle `credit_bonus_cashback(..., 'bonus_cashback_quest', period_identifier)`.
6. Si `quest.badge_type_id` défini : INSERT `user_badges` (ON CONFLICT DO NOTHING).
7. Si `quest.bonus_xp > 0` : `UPDATE profiles SET total_xp = COALESCE(total_xp,0) + bonus_xp`.
8. Si `quest.bonus_cashback > 0` : `UPDATE profiles SET cashback_balance = COALESCE(cashback_balance,0) + bonus_cashback`.
9. INSERT `quest_completion_logs`.
10. `UPDATE quest_progress SET status = 'rewarded', rewarded_at = now(), updated_at = now()`.

## Securite

Depuis la migration **040 (18/05/2026)** :

- Première instruction : `PERFORM public.assert_admin()`. Voir [`assert_admin.md`](./assert_admin.md).
- `GRANT EXECUTE TO authenticated`, `REVOKE EXECUTE FROM PUBLIC, anon`.
- L'audit trail est désormais non falsifiable : dérivé de `auth.uid()` au lieu de l'ex-paramètre `p_admin_id`.
- Le trigger automatique `distribute_quest_rewards()` (sans le `_reward` singulier) reste lui aussi protégé : il s'exécute en owner `postgres` (bypass automatique via `session_user`).

## Notes

- Idempotente via le check `status = completed` : appeler 2× ne distribue pas 2 fois.
- Pour rejouer après un échec partiel, repasser manuellement le statut à `completed` puis rappeler la RPC.
- Le `bonus_xp` / `bonus_cashback` met à jour `profiles.total_xp` et `profiles.cashback_balance` — colonnes legacy conservées pour rétro-compat même si le calcul officiel passe par la vue `user_stats`.
