# Function: get_current_xp_rank

Retourne le rang et le total d'XP d'un utilisateur précis dans le leaderboard de la période courante (semaine / mois / année). Retourne un tableau vide si l'utilisateur n'est pas classé.

Wrapper sécurisé des 3 MV `weekly_xp_leaderboard` / `monthly_xp_leaderboard` / `yearly_xp_leaderboard` — voir [`get_current_xp_leaderboard`](./get_current_xp_leaderboard.md) pour le pattern et le contexte.

## Signature

```sql
CREATE FUNCTION public.get_current_xp_rank(
  p_period_type TEXT,
  p_customer_id UUID
) RETURNS TABLE (rank BIGINT, total_xp BIGINT)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public, pg_temp
```

## Paramètres

| Paramètre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `p_period_type` | `TEXT` | Oui | - | `'weekly'`, `'monthly'` ou `'yearly'`. Toute autre valeur lève `SQLSTATE 22023`. |
| `p_customer_id` | `UUID` | Oui | - | ID utilisateur (`profiles.id`) |

## Retour

- Si l'utilisateur est classé : 1 ligne `(rank, total_xp)`.
- Sinon : tableau vide. Les consommateurs doivent traiter `data?.[0] ?? null`.

## Autorisations

- `GRANT EXECUTE TO anon, authenticated` — donnée publique (rang dans classement).

## Consommateurs

- `royaume-paraiges-front/src/features/gains/services/leaderboardService.ts` — méthodes `getUserWeeklyRank`, `getUserMonthlyRank`, `getUserYearlyRank` (helper privé `fetchUserRank`).
- `royaume-paraiges-admin/src/lib/services/userService.ts` — page `/users/[id]` (rang weekly/monthly/yearly affiché sur la fiche utilisateur).

## Notes

- **Bigint** : `rank` et `total_xp` sont des `bigint` ; appliquer `Number()` côté client.
- **Pas de `*_total_spent`** : exclu volontairement (règle produit « zéro euro côté client »).
- **Migration introductrice** : `security_revoke_xp_leaderboard_matviews` (18/05/2026).
- **XP = tous les gains** (migration `058_leaderboard_include_all_gains`, 03/07/2026) : `total_xp` inclut désormais les gains sans ticket (quêtes, bonus, actions utilisateur) — voir [`get_current_xp_leaderboard`](./get_current_xp_leaderboard.md).
