# Function: get_period_preview_public

Prévisualisation du leaderboard projeté + récompenses associées, **destinée au front client (Expo)** pour la page classement. Introduite par la migration **040 (Security Definer hardening, 18/05/2026)** comme équivalent public sûr de [`get_period_preview`](./get_period_preview.md) (qui reste admin-only).

Aucune écriture en BDD, aucun UUID exposé, aucun montant en euros côté admin trail (cohérent avec la règle produit « zéro euro côté client »).

## Signature

```sql
CREATE FUNCTION public.get_period_preview_public(
  p_period_type VARCHAR,
  p_period_identifier VARCHAR,
  p_top_n INTEGER DEFAULT 20
)
RETURNS TABLE (
  rank BIGINT,
  username TEXT,
  avatar_url TEXT,
  current_xp BIGINT,
  projected_reward_amount INTEGER,
  projected_reward_type TEXT,
  is_me BOOLEAN
)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
```

## Parametres

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `p_period_type` | `VARCHAR` | Oui | - | `'weekly'`, `'monthly'` ou `'yearly'`. Toute autre valeur lève `SQLSTATE 22023`. |
| `p_period_identifier` | `VARCHAR` | Oui | - | Identifiant de période (ex: `2026-W21`). Si `NULL`, fallback sur `get_period_identifier(p_period_type)` (période courante). |
| `p_top_n` | `INTEGER` | Non | `20` | Nombre de lignes renvoyées en tête de classement. La ligne de l'utilisateur connecté est toujours incluse, même si elle est hors top N. |

## Retour

Une ligne par utilisateur dans le top N (+ une ligne supplémentaire pour `auth.uid()` si hors top N), triées par rang croissant.

| Colonne | Description |
|---------|-------------|
| `rank` | Position dans le classement projeté |
| `username` | Pseudo affichable (jamais d'UUID, jamais d'email) |
| `avatar_url` | URL avatar (peut être `NULL`) |
| `current_xp` | XP cumulés sur la période |
| `projected_reward_amount` | Montant projeté du coupon (centimes) si tier fixe, sinon pourcentage si tier %, sinon `NULL` |
| `projected_reward_type` | `'coupon_amount'` / `'coupon_percentage'` / `NULL` |
| `is_me` | `true` si la ligne correspond à `auth.uid()`, `false` sinon |

## Logique

1. Valide `p_period_type` (lève `22023` si invalide).
2. Détermine le `period_identifier` effectif et la vue matérialisée correspondante (`weekly_xp_leaderboard` / `monthly_xp_leaderboard` / `yearly_xp_leaderboard`).
3. Charge les tiers : `period_reward_configs.custom_tiers` si défini pour la période, sinon `reward_tiers` actifs pour le `period_type` (triés par `display_order`).
4. CTE `lb` : recalcule le leaderboard depuis la vue matérialisée filtré `NOT p.is_test` (les MV ne filtrent pas elles-mêmes).
5. CTE `tier_lookup` : éclate le jsonb des tiers (`rank_from`, `rank_to`, `coupon_template_id`).
6. CTE `enriched` : associe chaque ligne du leaderboard à son tier via `rank BETWEEN rank_from AND rank_to`, joint `coupon_templates` pour récupérer `amount` / `percentage`.
7. SELECT final : joint `profiles` pour `username` / `avatar_url`, garde les rangs `<= p_top_n` OU la ligne de `auth.uid()`, sans **jamais** exposer le `customer_id`.

## Securite

- `SECURITY DEFINER` : bypass RLS pour la jointure sur `profiles` (lecture seule, colonnes minimales).
- `REVOKE EXECUTE FROM PUBLIC, anon` puis `GRANT EXECUTE TO authenticated` — uniquement les utilisateurs connectés.
- **Pas de guard `assert_*`** : la fonction est volontairement lisible par tout authentifié (c'est sa raison d'être). La sûreté repose sur le fait qu'aucun UUID/email/montant € sensible n'est exposé.
- Filtre `NOT p.is_test` appliqué pour exclure les comptes de test du classement public.

## Notes

- **Pas d'UUID** : `customer_id` n'est jamais exposé. Le client identifie sa propre ligne via `is_me = true`.
- **Pas d'écriture** : contrairement à `distribute_period_rewards_v2` (mode non-preview), aucune ligne n'est insérée dans `coupon_distribution_logs` / `period_reward_configs` / `coupons` / `gains` / `user_badges`.
- **Cohérence avec la grille de récompenses** : tant que les tiers en `reward_tiers` ne sont pas modifiés mid-période, l'utilisateur voit la projection réelle de ce qu'il toucherait à la clôture.
- **Limitation** : même bug que `distribute_period_rewards_v2` — la vue matérialisée `*_xp_leaderboard` ne contient que la période courante, donc le preview d'une période passée renvoie en réalité le classement courant. Acceptable pour le front (qui n'expose que la période courante côté UX).

## Voir aussi

- [`get_period_preview`](./get_period_preview.md) — équivalent admin-only renvoyant un JSONB plus riche (avec `customer_id` et `tier_name`)
- [`distribute_period_rewards_v2`](./distribute_period_rewards_v2.md) — distribution réelle
