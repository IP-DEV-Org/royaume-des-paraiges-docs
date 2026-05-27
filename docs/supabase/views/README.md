# Vues et Vues Materialisees

## Vues Materialisees (4)

Les vues materialisees stockent les resultats et doivent etre rafraichies periodiquement.


### monthly_xp_leaderboard

Leaderboard mensuel (1er au dernier jour du mois). Joindre avec profiles pour récupérer username, nom, etc. Exclut les comptes test (`profiles.is_test = true`).

```sql
 SELECT r.customer_id,
    COALESCE(sum(g.xp), 0::bigint) AS monthly_xp,
    count(DISTINCT r.id) AS monthly_receipt_count,
    count(DISTINCT r.establishment_id) AS monthly_establishment_count,
    COALESCE(sum(r.amount), 0::bigint) AS monthly_total_spent,
    min(r.created_at) AS first_receipt_at,
    row_number() OVER (ORDER BY (COALESCE(sum(g.xp), 0::bigint)) DESC, (min(r.created_at))) AS rank
   FROM receipts r
     JOIN profiles p ON p.id = r.customer_id
     LEFT JOIN gains g ON g.receipt_id = r.id
  WHERE r.created_at >= date_trunc('month'::text, now()) AND r.created_at < (date_trunc('month'::text, now()) + '1 mon'::interval)
    AND NOT p.is_test
  GROUP BY r.customer_id
 HAVING COALESCE(sum(g.xp), 0::bigint) > 0
  ORDER BY (row_number() OVER (ORDER BY (COALESCE(sum(g.xp), 0::bigint)) DESC, (min(r.created_at))));
```


### user_stats

Vue matérialisée combinant les statistiques XP et cashback par utilisateur.
Joint directement `profiles → gains` via `customer_id` (au lieu de passer par `receipts`), ce qui permet de comptabiliser les gains avec `receipt_id = NULL` (bonus cashback directs).

```sql
 SELECT p.id AS customer_id,
    COALESCE(sum(g.xp), 0::bigint) AS total_xp,
    COALESCE(sum(g.cashback_money), 0::bigint) AS cashback_earned,
    COALESCE(( SELECT sum(rl.amount) AS sum
           FROM receipt_lines rl
             JOIN receipts r_sub ON r_sub.id = rl.receipt_id
          WHERE r_sub.customer_id = p.id AND rl.payment_method = 'cashback'::payment_method), 0::bigint) AS cashback_spent,
    COALESCE(sum(g.cashback_money), 0::bigint) - COALESCE(( SELECT sum(rl.amount) AS sum
           FROM receipt_lines rl
             JOIN receipts r_sub ON r_sub.id = rl.receipt_id
          WHERE r_sub.customer_id = p.id AND rl.payment_method = 'cashback'::payment_method), 0::bigint) AS cashback_available
   FROM profiles p
     LEFT JOIN gains g ON g.customer_id = p.id
  GROUP BY p.id;
```


### weekly_xp_leaderboard

Leaderboard hebdomadaire (lundi-dimanche). Joindre avec profiles pour récupérer username, nom, etc. Exclut les comptes test (`profiles.is_test = true`).

```sql
 SELECT r.customer_id,
    COALESCE(sum(g.xp), 0::bigint) AS weekly_xp,
    count(DISTINCT r.id) AS weekly_receipt_count,
    count(DISTINCT r.establishment_id) AS weekly_establishment_count,
    COALESCE(sum(r.amount), 0::bigint) AS weekly_total_spent,
    min(r.created_at) AS first_receipt_at,
    row_number() OVER (ORDER BY (COALESCE(sum(g.xp), 0::bigint)) DESC, (min(r.created_at))) AS rank
   FROM receipts r
     JOIN profiles p ON p.id = r.customer_id
     LEFT JOIN gains g ON g.receipt_id = r.id
  WHERE r.created_at >= date_trunc('week'::text, now()) AND r.created_at < (date_trunc('week'::text, now()) + '7 days'::interval)
    AND NOT p.is_test
  GROUP BY r.customer_id
 HAVING COALESCE(sum(g.xp), 0::bigint) > 0
  ORDER BY (row_number() OVER (ORDER BY (COALESCE(sum(g.xp), 0::bigint)) DESC, (min(r.created_at))));
```


### yearly_xp_leaderboard

Leaderboard annuel (1er janvier au 31 décembre). Joindre avec profiles pour récupérer username, nom, etc. Exclut les comptes test (`profiles.is_test = true`).

```sql
 SELECT r.customer_id,
    COALESCE(sum(g.xp), 0::bigint) AS yearly_xp,
    count(DISTINCT r.id) AS yearly_receipt_count,
    count(DISTINCT r.establishment_id) AS yearly_establishment_count,
    COALESCE(sum(r.amount), 0::bigint) AS yearly_total_spent,
    min(r.created_at) AS first_receipt_at,
    row_number() OVER (ORDER BY (COALESCE(sum(g.xp), 0::bigint)) DESC, (min(r.created_at))) AS rank
   FROM receipts r
     JOIN profiles p ON p.id = r.customer_id
     LEFT JOIN gains g ON g.receipt_id = r.id
  WHERE r.created_at >= date_trunc('year'::text, now()) AND r.created_at < (date_trunc('year'::text, now()) + '1 year'::interval)
    AND NOT p.is_test
  GROUP BY r.customer_id
 HAVING COALESCE(sum(g.xp), 0::bigint) > 0
  ORDER BY (row_number() OVER (ORDER BY (COALESCE(sum(g.xp), 0::bigint)) DESC, (min(r.created_at))));
```


## Vues (6)


### avg_ticket_12m

Panier moyen des tickets sur les 12 derniers mois, hors comptes de test (`profiles.is_test = true`). Utilisé par la PR#4 (dashboard santé des quêtes) pour calculer le ratio des quêtes non-consommation (`orders_count`, `establishments_visited`) — les autres types de quêtes utilisent soit un prix de référence par `consumption_type` (table `admin_settings`), soit sont exclues du check (cas `xp_earned` et `quest_completed`).

```sql
 SELECT COALESCE(avg(r.amount), 0::numeric)::integer AS avg_ticket_cents,
    count(*) AS sample_size
   FROM receipts r
     JOIN profiles p ON p.id = r.customer_id
  WHERE r.created_at >= (now() - '12 mons'::interval)
    AND p.is_test IS NOT TRUE;
```


### reward_distribution_stats

Statistiques agrégées des distributions de récompenses par période (nombre de gagnants, coupons créés, montant distribué, badges attribués).

```sql
 SELECT pc.period_type,
    pc.period_identifier,
    pc.rewards_distributed_count,
    pc.status,
    pc.distribution_duration_ms,
    pc.closed_at,
    count(DISTINCT lrd.customer_id) AS unique_winners,
    count(c.id) AS total_coupons_created,
    sum(
        CASE
            WHEN c.amount IS NOT NULL THEN c.amount
            ELSE 0
        END) AS total_euros_distributed,
    count(ub.id) AS total_badges_awarded
   FROM period_closures pc
     LEFT JOIN leaderboard_reward_distributions lrd ON pc.period_type::text = lrd.period_type::text AND pc.period_identifier::text = lrd.period_identifier::text
     LEFT JOIN coupons c ON c.id = lrd.coupon_amount_id OR c.id = lrd.coupon_percentage_id
     LEFT JOIN user_badges ub ON ub.period_type::text = pc.period_type::text AND ub.period_identifier::text = pc.period_identifier::text
  GROUP BY pc.id, pc.period_type, pc.period_identifier, pc.rewards_distributed_count, pc.status, pc.distribution_duration_ms, pc.closed_at
  ORDER BY pc.closed_at DESC;
```


### public_profiles

Vue publique exposant uniquement les colonnes non-sensibles des profils. Utilisée pour les lookups publics (classements, profils sociaux).

```sql
 SELECT id,
    username,
    avatar_url,
    attached_establishment_id
   FROM profiles;
```


### cashpad_health_stats_30d

Statistiques de santé du matching Cashpad par établissement sur les 30 derniers jours. Agrège : nombre de matchés/ambigus/orphelins/exclus, taux de match, confiance moyenne, et paramètres adaptatifs en cours.

```sql
 WITH bounds AS (
         SELECT (now() - '30 days'::interval) AS since
        )
 SELECT e.id AS establishment_id,
    e.title AS establishment_title,
    e.cashpad_installation_id,
    count(*) FILTER (WHERE (rec.status = 'matched')) AS n_matched,
    count(*) FILTER (WHERE (rec.status = 'ambiguous')) AS n_ambiguous,
    count(*) FILTER (WHERE (rec.status = 'orphan_royaume')) AS n_orphan,
    count(*) FILTER (WHERE (rec.status = 'excluded_cashback')) AS n_excluded,
    count(*) FILTER (WHERE (rec.status = 'orphan_royaume' AND rec.cancelled_match_id IS NOT NULL)) AS n_orphan_cancelled_match,
    count(*) AS n_total,
    round((100.0 * count(*) FILTER (WHERE rec.status = 'matched')::numeric)
      / NULLIF(count(*) FILTER (WHERE rec.status IN ('matched','ambiguous','orphan_royaume')), 0)::numeric, 1) AS match_rate_pct,
    round(avg(rec.confidence_score) FILTER (WHERE rec.status = 'matched'), 1) AS avg_confidence,
    p.window_seconds AS current_window_s,
    p.clock_offset_seconds AS current_offset_s,
    p.sample_size AS params_sample_size,
    p.computed_at AS params_computed_at
   FROM establishments e
     LEFT JOIN receipts r ON r.establishment_id = e.id AND r.created_at >= (SELECT since FROM bounds)
     LEFT JOIN cashpad_reconciliations rec ON rec.receipt_id = r.id
     LEFT JOIN cashpad_matching_params p ON p.establishment_id = e.id
  WHERE e.cashpad_installation_id IS NOT NULL
  GROUP BY e.id, e.title, e.cashpad_installation_id, p.window_seconds, p.clock_offset_seconds, p.sample_size, p.computed_at
  ORDER BY e.id;
```


### cashpad_clock_drift

Détecte la dérive d'horloge entre Cashpad et Royaume par établissement. Compare la médiane du delta temporel sur 7 jours vs 30 jours pour signaler les dérives récentes.

```sql
 WITH recent AS (
         SELECT r.establishment_id,
            percentile_cont(0.5) WITHIN GROUP (ORDER BY rec.time_delta_seconds::double precision) AS median_7d,
            count(*) AS n_7d
           FROM cashpad_reconciliations rec
             JOIN receipts r ON r.id = rec.receipt_id
          WHERE rec.status = 'matched' AND rec.time_delta_seconds IS NOT NULL
            AND rec.reconciled_at >= (now() - '7 days'::interval)
          GROUP BY r.establishment_id
        ), baseline AS (
         SELECT r.establishment_id,
            percentile_cont(0.5) WITHIN GROUP (ORDER BY rec.time_delta_seconds::double precision) AS median_30d,
            count(*) AS n_30d
           FROM cashpad_reconciliations rec
             JOIN receipts r ON r.id = rec.receipt_id
          WHERE rec.status = 'matched' AND rec.time_delta_seconds IS NOT NULL
            AND rec.reconciled_at >= (now() - '30 days'::interval)
          GROUP BY r.establishment_id
        )
 SELECT b.establishment_id,
    round(b.median_30d) AS median_30d_s,
    round(r.median_7d) AS median_7d_s,
    round(r.median_7d - b.median_30d) AS drift_s,
    r.n_7d,
    b.n_30d
   FROM baseline b
     LEFT JOIN recent r ON r.establishment_id = b.establishment_id;
```


### cashpad_window_feedback

Feedback loop des liens manuels pour ajuster la fenêtre de matching. Signale quand la fenêtre actuelle est trop serrée (liens manuels hors fenêtre) et propose un p95 incluant les liens manuels.

```sql
 SELECT r.establishment_id,
    count(*) FILTER (WHERE rec.manual_link_delta_seconds IS NOT NULL) AS manual_links_total,
    count(*) FILTER (WHERE rec.manual_link_delta_seconds IS NOT NULL
      AND abs(rec.manual_link_delta_seconds - COALESCE(p.clock_offset_seconds, 0)) > COALESCE(p.window_seconds, 200)) AS manual_links_out_of_window,
    COALESCE(p.window_seconds, 200) AS current_window_s,
    COALESCE(p.clock_offset_seconds, 0) AS current_offset_s,
    round(percentile_cont(0.95) WITHIN GROUP (ORDER BY
      abs(COALESCE(rec.manual_link_delta_seconds, rec.time_delta_seconds) - COALESCE(p.clock_offset_seconds, 0))::double precision
    ))::integer AS suggested_window_p95
   FROM cashpad_reconciliations rec
     JOIN receipts r ON r.id = rec.receipt_id
     LEFT JOIN cashpad_matching_params p ON p.establishment_id = r.establishment_id
  WHERE rec.status IN ('matched', 'ambiguous') OR rec.manual_link_delta_seconds IS NOT NULL
  GROUP BY r.establishment_id, p.window_seconds, p.clock_offset_seconds;
```

