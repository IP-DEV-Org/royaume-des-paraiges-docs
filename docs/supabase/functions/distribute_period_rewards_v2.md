# Function: distribute_period_rewards_v2

Distribue les recompenses du leaderboard pour une periode donnee. Supporte la previsualisation, les paliers configurables et le systeme bonus cashback.

## Signature

```sql
CREATE FUNCTION distribute_period_rewards_v2(
  p_period_type VARCHAR,
  p_period_identifier VARCHAR DEFAULT NULL,
  p_preview_only BOOLEAN DEFAULT false,
  p_force BOOLEAN DEFAULT false
) RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public'
```

## Parametres

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `p_period_type` | `VARCHAR` | Oui | - | Type de periode: weekly, monthly, yearly |
| `p_period_identifier` | `VARCHAR` | Non | `NULL` | Identifiant de periode (ex: 2026-W06). Si NULL, utilise la periode courante. **Bug connu** : ce paramètre est lu pour le scope des `period_reward_configs` et le `period_identifier` stocké, mais la sélection des utilisateurs lit la vue matérialisée `*_xp_leaderboard` telle quelle, qui ne contient que la période courante — la distribution / le preview d'une période passée renvoie donc le leaderboard du moment. À corriger dans une migration ultérieure. |
| `p_preview_only` | `BOOLEAN` | Non | `false` | Si true, retourne un apercu sans distribuer |
| `p_force` | `BOOLEAN` | Non | `false` | Si true, force la distribution meme si deja effectuee |

> **Migration 040 (18/05/2026)** : le paramètre `p_admin_id` a été **retiré**. L'audit trail (`period_reward_configs.distributed_by`, `coupon_distribution_logs.distributed_by`) est désormais dérivé de `auth.uid()` côté serveur.

## Retour

### Mode preview (`p_preview_only = true`)

```json
{
  "success": true,
  "preview": true,
  "period_type": "weekly",
  "period_identifier": "2026-W06",
  "rewards_to_distribute": 3,
  "data": [
    {
      "customer_id": "uuid",
      "rank": 1,
      "xp": 150,
      "tier_name": "Top 1",
      "coupon_amount": 500,
      "coupon_percentage": null,
      "badge_type_id": 1
    }
  ]
}
```

### Mode distribution

```json
{
  "success": true,
  "period_type": "weekly",
  "period_identifier": "2026-W06",
  "rewards_distributed": 3,
  "duration_ms": 125,
  "errors": []
}
```

## Logique

1. `PERFORM public.assert_admin()` — bloque les appelants non-admin (voir Securite)
2. Valide le `period_type` (weekly/monthly/yearly)
3. Determine le `period_identifier` si non fourni
4. Verifie l'idempotence : rejette si deja distribue (sauf `p_force=true`)
5. Charge les paliers : `period_reward_configs.custom_tiers` ou `reward_tiers` par defaut
6. Parcourt le leaderboard (vue materialisee) et associe chaque utilisateur a un palier
7. Pour chaque recompense :
   - Cree un coupon (amount ou percentage depuis le template)
   - Si montant fixe (bonus cashback) : `used=true`, appelle `credit_bonus_cashback()` avec `period_identifier`
   - Si badge configure : insere dans `user_badges`
   - Log dans `coupon_distribution_logs` avec `distributed_by = auth.uid()`
8. Met a jour `period_reward_configs` avec `status='distributed'`, `distributed_by = auth.uid()`

## Securite

Depuis la migration **040 (Security Definer hardening, 18/05/2026)** :

- Première instruction : `PERFORM public.assert_admin()`. Voir [`assert_admin.md`](./assert_admin.md).
- Bypass automatique pour `service_role` et `session_user IN (postgres, supabase_admin)` (cas des pg_cron jobs de distribution).
- `GRANT EXECUTE TO authenticated`, `REVOKE EXECUTE FROM PUBLIC, anon`.
- Audit trail (`distributed_by`) non falsifiable : dérivé de `auth.uid()` au lieu de l'ex-paramètre `p_admin_id`.

## Vues materialisees utilisees

| period_type | Vue |
|-------------|-----|
| weekly | `weekly_xp_leaderboard` |
| monthly | `monthly_xp_leaderboard` |
| yearly | `yearly_xp_leaderboard` |

## Exemple

```sql
-- Previsualisation
SELECT distribute_period_rewards_v2(
  'weekly',
  '2026-W06',
  p_preview_only := true
);

-- Distribution
SELECT distribute_period_rewards_v2(
  'weekly',
  '2026-W06'
);

-- Force re-distribution
SELECT distribute_period_rewards_v2(
  'weekly',
  '2026-W06',
  p_force := true
);
```

## Notes

- Remplace `distribute_leaderboard_rewards()` (legacy)
- Idempotente par defaut via `period_reward_configs`
- Les erreurs individuelles n'arretent pas la distribution globale
- Les coupons montant fixe sont immediatement convertis en bonus cashback
