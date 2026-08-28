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
| `p_period_identifier` | `VARCHAR` | Non | `NULL` | Identifiant de periode (ex: 2026-W06). Si NULL, vise la **période close** via [`get_previous_period_identifier`](./get_previous_period_identifier.md) (fonction préexistante, réécrite par la 081c) — c'est le cas des trois crons, déclenchés juste après la bascule de période. Un identifiant mal formé est rejeté (`success: false`) au lieu d'être silencieusement ignoré. |
| `p_preview_only` | `BOOLEAN` | Non | `false` | Si true, retourne un apercu sans distribuer |
| `p_force` | `BOOLEAN` | Non | `false` | Si true, force la distribution meme si deja effectuee |

> **Migration 040 (18/05/2026)** : le paramètre `p_admin_id` a été **retiré**. L'audit trail (`period_reward_configs.distributed_by`, `coupon_distribution_logs.distributed_by`) est désormais dérivé de `auth.uid()` côté serveur.

> **Migration 081 (28/08/2026)** : la fonction lit enfin la période qu'on lui demande. Deux défauts corrigés ensemble :
>
> 1. **La source ignorait la période.** Le classement venait des vues matérialisées `*_xp_leaderboard`, filtrées sur `date_trunc(..., now())` : la période y est câblée sur l'instant du dernier `REFRESH` et une matview ne peut pas prendre de paramètre. `p_period_identifier` ne servait qu'à étiqueter les lignes écrites.
> 2. **Le défaut visait la période qui s'ouvre.** `get_period_identifier()` résout la période *en cours* ; les crons partent à 00:05 le lundi, 00:10 le 1er du mois et 00:15 le 1er janvier (base en UTC), quand la période en cours a cinq minutes d'existence.
>
> Ce n'était pas théorique : `period_reward_configs` porte 9 lignes hebdo + 2 mensuelles en statut `distributed` entre janvier et avril 2026, étiquetées de la période qui *s'ouvrait* (le cron du lundi 27/04 a écrit `2026-W18`, la semaine close étant W17), pendant que `coupon_distribution_logs` ne contient que 3 entrées manuelles de mai 2026 et qu'aucun coupon `leaderboard_*` n'existe. Ces distributions ont tourné pour de vrai et n'ont récompensé personne. La désactivation des paliers en mai 2026 a ensuite masqué le défaut (sortie anticipée sur « No reward tiers configured »).
>
> **Rattrapage** : les semaines de janvier à avril 2026 portent déjà une ligne `distributed` (mal étiquetée), donc une re-distribution sur ces périodes exige `p_force := true`.

## Retour

### Mode preview (`p_preview_only = true`)

```json
{
  "success": true,
  "preview": true,
  "period_type": "weekly",
  "period_identifier": "2026-W06",
  "period_start": "2026-02-02T00:00:00+00:00",
  "period_end": "2026-02-09T00:00:00+00:00",
  "period_closed": true,
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
  "period_start": "2026-02-02T00:00:00+00:00",
  "period_end": "2026-02-09T00:00:00+00:00",
  "period_closed": true,
  "rewards_distributed": 3,
  "duration_ms": 125,
  "errors": []
}
```

## Logique

1. `PERFORM public.assert_admin()` — bloque les appelants non-admin (voir Securite)
2. Valide le `period_type` (weekly/monthly/yearly)
3. Determine le `period_identifier` si non fourni : **periode close** (`get_previous_period_identifier`)
4. Resout les bornes via `get_period_bounds` — valide l'identifiant et alimente `period_start` / `period_end` du retour
5. Verifie l'idempotence : rejette si deja distribue (sauf `p_force=true`)
6. Charge les paliers : `period_reward_configs.custom_tiers` ou `reward_tiers` par defaut
7. Fige le classement de la periode via `get_period_leaderboard` (pagination jusqu'au plus haut `rank_to` du bareme) et associe chaque utilisateur a un palier
8. Pour chaque recompense :
   - Cree un coupon (amount ou percentage depuis le template)
   - Si montant fixe (bonus cashback) : `used=true`, appelle `credit_bonus_cashback()` avec `period_identifier`
   - Si badge configure : insere dans `user_badges`
   - Log dans `coupon_distribution_logs` avec `distributed_by = auth.uid()`
9. Met a jour `period_reward_configs` avec `status='distributed'`, `distributed_by = auth.uid()`

## Securite

Depuis la migration **040 (Security Definer hardening, 18/05/2026)** :

- Première instruction : `PERFORM public.assert_admin()`. Voir [`assert_admin.md`](./assert_admin.md).
- Bypass automatique pour `service_role` et `session_user IN (postgres, supabase_admin)` (cas des pg_cron jobs de distribution).
- `GRANT EXECUTE TO authenticated`, `REVOKE EXECUTE FROM PUBLIC, anon`.
- Audit trail (`distributed_by`) non falsifiable : dérivé de `auth.uid()` au lieu de l'ex-paramètre `p_admin_id`.

> **Migrations 083 et 084 (28/08/2026)** : deux défauts supplémentaires, découverts en lançant le rattrapage de l'historique.
>
> - **084 — l'INSERT de coupon référençait `coupons.establishment_id`, colonne qui n'existe pas** (piège documenté dans le CLAUDE.md du projet). Chaque récompense en argent levait `42703`, erreur avalée par le bloc `EXCEPTION` de la boucle : la fonction retournait `success: true`, la ligne partait en `failed` dans `coupon_distribution_logs`, et rien ne remontait. **Aucune récompense en argent du classement n'a jamais pu être créée**, même une fois le classement réparé par la 081.
> - **083 — un palier avec un badge mais sans `coupon_template_id` n'attribuait rien**, pas même son badge : tout le bloc de récompense était conditionné à l'existence d'un coupon. Concerne « Top 10 de la semaine » (rangs 4-10) et « Podium / Top 10 de l'année ». Le badge est désormais attribué indépendamment du coupon — ce qui rend possible une configuration **badges seuls**, paliers actifs avec `coupon_template_id` à `NULL`. **C'est la configuration en vigueur depuis la migration 085** : le classement récompense en badges, l'argent est écarté jusqu'à nouvel ordre. Attention, couper l'argent en passant les paliers `is_active = false` couperait aussi les badges — la fonction sort sur « No reward tiers configured » avant même de lire le classement.
>
> La 083 ajoute par ailleurs le journal [`reward_distribution_runs`](../tables/reward_distribution_runs.md) et cesse de marquer `period_reward_configs` en `distributed` quand **aucune** récompense n'est passée : une période qui n'a rien distribué reste rejouable sans `p_force`.

## Source du classement

Depuis la migration 081 : [`get_period_leaderboard(p_period_type, p_period_identifier, p_limit, p_offset)`](./get_period_leaderboard.md), pour les trois périodicités. C'est la même source que `build_report_leaderboard` (rapports e-mail) et [`get_period_rank`](./get_period_rank.md) : mêmes exclusions (`is_test`, `deleted_at`) et même départage (XP décroissant, puis premier ticket / premier gain).

`p_limit` étant borné à 100 (migration 072), le classement est lu **par pages de 100 jusqu'au plus haut `rank_to` du barème**, puis figé en JSONB avant la boucle de distribution. Sans cette pagination, un palier configuré au-delà du rang 100 aurait été tronqué en silence.

Les vues matérialisées `weekly_xp_leaderboard` / `monthly_` / `yearly_` **ne sont plus lues ici** — mais elles restent en place : rafraîchies toutes les 5 min par le cron `refresh-leaderboard-views`, elles alimentent `get_current_xp_leaderboard`, `get_current_xp_rank` et `create_receipt`, c'est-à-dire le classement **temps réel** de l'app cliente, où « la période en cours » est la bonne réponse.

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

- Remplace `distribute_leaderboard_rewards()` (legacy — qui, lui, lit toujours les vues matérialisées et ignore la période ; aucun cron ni projet applicatif ne l'appelle)
- Idempotente par defaut via `period_reward_configs`
- Les erreurs individuelles n'arretent pas la distribution globale
- Les coupons montant fixe sont immediatement convertis en bonus cashback
