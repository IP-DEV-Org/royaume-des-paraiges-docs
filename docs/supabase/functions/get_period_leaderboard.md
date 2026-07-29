# Function: get_period_leaderboard

Retourne le classement XP d'une **période donnée** (passée ou courante), identifiée par son `period_identifier`. Contrairement à [`get_current_xp_leaderboard`](./get_current_xp_leaderboard.md) qui lit des vues matérialisées figées sur `now()`, cette fonction **recalcule à la volée** depuis `gains`, ce qui permet de consulter n'importe quelle période close.

Complétée par [`get_period_rank`](./get_period_rank.md) pour obtenir le rang d'un joueur hors du top affiché.

## Signature

```sql
CREATE FUNCTION public.get_period_leaderboard(
  p_period_type CHARACTER VARYING,
  p_period_identifier CHARACTER VARYING,
  p_limit INTEGER DEFAULT 50,
  p_offset INTEGER DEFAULT 0
) RETURNS TABLE (
  customer_id UUID, username TEXT, avatar_url TEXT,
  total_xp BIGINT, receipt_count BIGINT, rank BIGINT, total_count BIGINT
)
LANGUAGE plpgsql
STABLE SECURITY DEFINER
SET search_path = public, pg_temp
```

## Paramètres

| Paramètre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `p_period_type` | `VARCHAR` | Oui | - | `'weekly'`, `'monthly'` ou `'yearly'` |
| `p_period_identifier` | `VARCHAR` | Oui | - | `2026-W29`, `2026-06`, `2026` (format `get_period_identifier`, semaines ISO) |
| `p_limit` | `INTEGER` | Non | 50 | **Borné à 100** depuis la migration 072 (voir Sécurité) |
| `p_offset` | `INTEGER` | Non | 0 | Négatif ramené à 0 |

## Retour

Une ligne par joueur classé, triée par rang. `total_count` répète sur chaque ligne l'effectif **total** de la période (non affecté par `p_limit`), ce qui permet de calculer la pagination.

Exclut : les profils `is_test`, les comptes supprimés (`deleted_at IS NOT NULL`), et les joueurs à 0 XP sur la période (`HAVING SUM(xp) > 0`).

Départage à XP égale : `COALESCE(first_receipt_at, first_gain_at)` — le premier arrivé prend le meilleur rang.

## Autorisations

- `GRANT EXECUTE TO authenticated, service_role` ; `REVOKE FROM PUBLIC, anon`.

## Sécurité — migration 072 (29/07/2026)

Jusqu'à cette migration, la fonction conservait le `GRANT TO PUBLIC` par défaut de PostgreSQL (jamais révoqué, et `CREATE OR REPLACE` préserve l'ACL) **et** acceptait un `p_limit` non borné. Un visiteur non authentifié pouvait donc extraire en un seul appel l'intégralité d'un classement — `customer_id` (UUID de compte), pseudo, avatar, XP, nombre de tickets — pour toute période depuis février 2026. La clé `anon` étant publique par construction (embarquée dans le bundle PWA), la surface était ouverte à quiconque.

C'était incohérent avec la doctrine appliquée à [`get_period_preview_public`](./get_period_preview_public.md), restreinte à `authenticated` précisément pour ne pas exposer d'UUID.

Deux correctifs : `REVOKE FROM PUBLIC, anon` (aucune route publique de l'app cliente n'affiche de classement — seuls les groupes `(auth)` et `legal` sont accessibles sans session) et plafond `LEAST(GREATEST(COALESCE(p_limit,50),1),100)`.

## Consommateurs

- `royaume-paraiges-front/src/features/gains/services/leaderboardService.ts` — méthode `getPeriodLeaderboard`, utilisée par le mode historique de `/leaderboard/[period]` (top 50 d'une période close).

## Notes

- ⚠️ **Le fichier de migration `051_leaderboards_exclude_deleted_users.sql` du repo est périmé** : il montre une agrégation depuis `receipts LEFT JOIN gains`, alors que la production agrège depuis `gains` (refonte 058 `leaderboard_include_all_gains`, appliquée sans être versionnée). Toujours lire la définition réelle via `SELECT pg_get_functiondef(oid) FROM pg_proc` avant de modifier. La migration 072 repart de la définition de prod.
- **Pas de snapshot** : le classement d'une période passée est recalculé à chaque appel. Une correction rétroactive (rollback, anonymisation RGPD, passage `is_test`) modifie donc un classement déjà clos — comportement voulu pour le RGPD.
- **Bigint** : tous les compteurs sont des `bigint`, appliquer `Number()` côté client.
- **Perf** : agrégation de la fenêtre de période à chaque appel, sans index sur `gains(created_at)`. OK aux volumes actuels (~5 k gains).
