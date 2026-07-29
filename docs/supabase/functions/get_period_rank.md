# Function: get_period_rank

Retourne le rang et le total d'XP d'un utilisateur précis dans le classement d'une **période passée** (identifiée par `period_identifier`). Complément de [`get_current_xp_rank`](./get_current_xp_rank.md), qui ne couvre que la période courante (vues matérialisées figées sur `now()`).

Miroir **exact** de la CTE de [`get_period_leaderboard`](./get_period_leaderboard.md) (qui liste le classement d'une période à la volée depuis `gains`) : mêmes bornes (`get_period_bounds`), mêmes filtres (`NOT is_test`, `deleted_at IS NULL`, `HAVING SUM(xp) > 0`), même tri (`total_xp DESC` puis `COALESCE(first_receipt_at, first_gain_at)`). **Toute évolution de `get_period_leaderboard` doit être répercutée ici**, sinon le rang divergera de la liste affichée.

⚠️ Ce « miroir exact » s'entend face à la définition **en production**, pas au fichier `051` du repo qui est périmé (voir les notes de `get_period_leaderboard`). Équivalence vérifiée empiriquement le 29/07/2026 : populations identiques (190/190 sur `2026-W29`, 307/307 sur `2026-06`), dont 58 joueurs de juin ayant des gains sans ticket, tous présents dans les deux.

## Signature

```sql
CREATE FUNCTION public.get_period_rank(
  p_period_type CHARACTER VARYING,
  p_period_identifier CHARACTER VARYING,
  p_customer_id UUID
) RETURNS TABLE (rank BIGINT, total_xp BIGINT)
LANGUAGE plpgsql
STABLE SECURITY DEFINER
SET search_path = public, pg_temp
```

## Paramètres

| Paramètre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `p_period_type` | `VARCHAR` | Oui | - | `'weekly'`, `'monthly'` ou `'yearly'`. Autre valeur : exception de `get_period_bounds`. |
| `p_period_identifier` | `VARCHAR` | Oui | - | Identifiant de période (`2026-W29`, `2026-06`, `2026`), format `get_period_identifier` (semaines ISO). |
| `p_customer_id` | `UUID` | Oui | - | ID utilisateur (`profiles.id`) |

## Retour

- Si l'utilisateur est classé sur la période : 1 ligne `(rank, total_xp)`.
- Sinon : tableau vide. Les consommateurs doivent traiter `data?.[0] ?? null`.

## Autorisations

- `GRANT EXECUTE TO authenticated, service_role` ; `REVOKE FROM PUBLIC, anon` (la page classement du front est derrière auth).

## Consommateurs

- `royaume-paraiges-front/src/features/gains/services/leaderboardService.ts` : méthode `getPeriodRank` (mode historique de la page classement, bannière « C'EST VOUS » quand le joueur est hors du top 50 affiché).

## Notes

- **Pas de snapshot** : le classement d'une période passée est recalculé à la volée depuis `gains` ; une correction rétroactive (rollback, anonymisation RGPD, passage `is_test`) modifie le rang retourné.
- **Bigint** : `rank` et `total_xp` sont des `bigint` ; appliquer `Number()` côté client.
- **Perf** : agrégation de la fenêtre de période à chaque appel, sans index sur `created_at`. OK aux volumes actuels (~5 k gains) ; créer `gains(created_at)` et `receipts(created_at)` si les volumes explosent.
- **Migration introductrice** : `071_get_period_rank` (29/07/2026).
