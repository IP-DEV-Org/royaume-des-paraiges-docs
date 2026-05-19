# Function: get_period_preview

Prévisualisation de la distribution des récompenses d'une période sans rien insérer en BDD. Wrapper admin-only autour de `distribute_period_rewards_v2(..., p_preview_only := true)`.

Admin-only depuis la migration **040 (Security Definer hardening, 18/05/2026)**.

## Signature

```sql
CREATE FUNCTION public.get_period_preview(
  p_period_type VARCHAR,
  p_period_identifier VARCHAR DEFAULT NULL
) RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
```

## Parametres

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `p_period_type` | `VARCHAR` | Oui | - | `'weekly'` / `'monthly'` / `'yearly'` |
| `p_period_identifier` | `VARCHAR` | Non | `NULL` | Identifiant de période (ex: `2026-W21`). Si `NULL`, période courante. Même bug que `distribute_period_rewards_v2` : le `period_identifier` n'est pas réellement utilisé pour scoper la lecture du leaderboard (la MV ne contient que la période courante). |

## Retour

Identique au mode preview de `distribute_period_rewards_v2` :

```json
{
  "success": true,
  "preview": true,
  "period_type": "weekly",
  "period_identifier": "2026-W21",
  "rewards_to_distribute": 3,
  "data": [
    {
      "customer_id": "uuid",
      "rank": 1,
      "xp": 150,
      "tier_name": "Champion Hebdomadaire 5€",
      "coupon_amount": 500,
      "coupon_percentage": null,
      "badge_type_id": 1
    }
  ]
}
```

## Logique

1. `PERFORM public.assert_admin()` — bloque les non-admins (voir Securite).
2. Appelle `distribute_period_rewards_v2(p_period_type, p_period_identifier, p_preview_only := true, p_force := false)` et retourne directement son résultat.

## Securite

Depuis la migration **040 (18/05/2026)** :

- Première instruction : `PERFORM public.assert_admin()`. Voir [`assert_admin.md`](./assert_admin.md).
- `REVOKE EXECUTE FROM PUBLIC, anon`, `GRANT EXECUTE TO authenticated` (filtré par le guard).
- Pour les besoins du front client (page classement), utiliser à la place [`get_period_preview_public`](./get_period_preview_public.md) : équivalent sans UUID exposé, ouvert à `authenticated`, retourne `username` + `avatar_url` + montant projeté.

## Notes

- Wrapper léger : toute la logique vit dans `distribute_period_rewards_v2`.
- Appelé depuis l'admin via `rewardService.previewPeriodRewards()`.
