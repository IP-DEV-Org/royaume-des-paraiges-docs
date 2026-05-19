# Function: get_coupon_stats

Retourne des statistiques agrégées sur tous les coupons : compteurs (actifs / utilisés / expirés), valeur totale active en centimes, répartition par type de distribution et par période.

Admin-only depuis la migration **040 (Security Definer hardening, 18/05/2026)**.

## Signature

```sql
CREATE FUNCTION public.get_coupon_stats() RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
```

## Parametres

Aucun.

## Retour

```json
{
  "total_coupons": 142,
  "total_active": 17,
  "total_used": 120,
  "total_expired": 5,
  "total_value_active_cents": 8500,
  "by_distribution_type": {
    "manual": 32,
    "leaderboard_weekly": 80,
    "leaderboard_monthly": 22,
    "quest": 8
  },
  "by_period": [
    { "period": "2026-W20", "count": 8, "used": 7 },
    { "period": "2026-W19", "count": 10, "used": 10 }
  ]
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `total_coupons` | INTEGER | Nombre total de coupons en BDD |
| `total_active` | INTEGER | `used = false AND (expires_at IS NULL OR expires_at > now())` |
| `total_used` | INTEGER | `used = true` |
| `total_expired` | INTEGER | `used = false AND expires_at <= now()` |
| `total_value_active_cents` | INTEGER | Somme des `amount` des coupons actifs (centimes). Les coupons % ne sont pas comptés (amount = NULL). |
| `by_distribution_type` | JSONB | Histogramme `distribution_type → count` |
| `by_period` | JSONB[] | Top 20 périodes (DESC) avec compteurs `count` / `used` |

## Logique

1. `PERFORM public.assert_admin()` — bloque les non-admins (voir Securite).
2. Single `SELECT jsonb_build_object(...)` sur la table `coupons` avec sub-queries pour `by_distribution_type` et `by_period`.

## Securite

Depuis la migration **040 (18/05/2026)** :

- Première instruction : `PERFORM public.assert_admin()`. Voir [`assert_admin.md`](./assert_admin.md).
- `SECURITY DEFINER` (lecture de `coupons` hors RLS).
- Aucun GRANT supplémentaire imposé par la migration — la sûreté repose sur le guard.

## Notes

- Affichée sur le dashboard admin / page coupons.
- Pas paginée (20 périodes max) : suffisant pour un dashboard.
