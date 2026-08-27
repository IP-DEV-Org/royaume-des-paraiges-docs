# Fonction : `get_email_report_payload`

Point d'entrée unique des données des **rapports e-mail** (migration 077, août 2026 ; étendu par la **migration 080**). Résout la période visée, dispatche vers le builder correspondant au `report_type` et renvoie un payload prêt à rendre. Toute la logique métier des rapports vit ici, en SQL : l'Edge Function `send-email-reports` ne fait que du rendu HTML et de l'envoi.

```sql
get_email_report_payload(
  p_report_key        TEXT,
  p_period_identifier TEXT DEFAULT NULL
) RETURNS JSONB
```

- `SECURITY DEFINER` + `assert_admin()` → accessible aux **admins** et au **service_role** (donc à l'Edge Function et au cron).
- Sans `p_period_identifier`, la période visée dépend de la **portée du rapport** (`email_reports.period_scope`, migration 079) :
  - `previous` (défaut) → **période écoulée** (`get_previous_period_identifier`). C'est le cas des bilans : un bilan sur une période en cours donnerait des chiffres partiels.
  - `current` → **période en cours** (`get_period_identifier`). C'est le cas des annonces (« les défis de la semaine ») : elles n'ont d'intérêt qu'avant que la période soit finie.
- Erreur `P0425` (`EMAIL_REPORT_UNKNOWN`) si la clé de rapport n'existe pas, `EMAIL_REPORT_TYPE_UNSUPPORTED` si le type n'a pas de builder.
- `EXECUTE` révoqué à `PUBLIC` et `anon`, accordé à `authenticated` et `service_role`.

## Forme du retour

```jsonc
{
  "report": { "key", "name", "report_type", "period_type", "period_scope", "subject" },
  "period": { "identifier", "label", "start", "end" },
  "data":   { /* dépend du report_type, voir builders */ },
  "generated_at": "2026-08-20T08:34:42Z"
}
```

`report.subject` est déjà interpolé : `{{period_label}}` du `subject_template` y est remplacé par le libellé humain.

## Builders

Les trois builders sont `SECURITY DEFINER` et **révoqués à `PUBLIC`, `anon` et `authenticated`** : ils ne sont appelables que depuis le dispatcher.

### `build_report_activity_summary(p_period_type, p_period_identifier)`

```jsonc
"data": {
  "previous_period": { "identifier", "label" },
  "activity": {
    "revenue_cents", "revenue_cents_prev",
    "receipts_count", "receipts_count_prev",
    "average_basket_cents", "average_basket_cents_prev",
    "by_establishment": [ { "id", "title", "revenue_cents", "receipts_count" } ]
  },
  "community": {
    "new_members", "new_members_prev",
    "total_members",
    "active_players", "active_players_prev",
    "level_up_players", "levels_gained",
    "by_rank": [ { "name", "sort_order", "players" } ]
  }
}
```

Règles de calcul :

- **Montants en centimes** (`receipts.amount`), comme partout ailleurs côté Supabase.
- **Comptes de test toujours exclus** (`profiles.is_test`).
- **Comptes supprimés** (`deleted_at`) exclus des métriques de communauté et des classements, mais **pas du chiffre d'affaires** : l'argent encaissé reste encaissé. Même arbitrage que `getSalesTotal` côté dashboard admin.
- La **période précédente** est celle qui contient la veille du début de période : donc le mois précédent, ou la semaine ISO précédente.
- **Montées de niveau** : niveau en fin de période vs niveau en début, via `compute_level_from_xp` appliqué à l'XP cumulé **depuis le début de saison** (année civile, cf. `get_season_xp`). `level_up_players` = nombre de joueurs ayant gagné au moins un niveau ; `levels_gained` = total des niveaux gagnés.
- **`by_rank`** ne couvre que les joueurs ayant gagné de l'XP **sur la saison en cours** à la fin de la période : ce n'est pas la répartition de tous les inscrits. Borne haute des rangs via `COALESCE(ranks.max_level, 9999)`.

### `build_report_leaderboard(p_period_type, p_period_identifier, p_top_n)`

```jsonc
"data": {
  "leaderboard": {
    "participants": 412,
    "total_xp": 49625,
    "top": [ { "rank", "username", "total_xp", "receipt_count" } ]
  }
}
```

- S'appuie sur [`get_period_leaderboard`](./get_period_leaderboard.md), qui exclut déjà comptes de test et supprimés et fournit `total_count` (= `participants`).
- `p_top_n` borné à [1, 100] ; défaut 10, surchargeable par `email_reports.options->>'top_n'`.
- **Aucun UUID exposé** : seuls pseudo, rang, XP et nombre de tickets sortent. `username` NULL est rendu « Joueur anonyme ».
- `total_xp` couvre **toute la période**, pas seulement le top N.

### `build_report_new_quests(p_period_type, p_period_identifier)`

Migration 080. Défis (quêtes récurrentes) **ouverts sur la période visée**, séparés en nouveautés et défis reconduits.

```jsonc
"data": {
  "previous_period": { "identifier", "label" },
  "quests": {
    "new":     [ /* défis qui s'ouvrent */ ],
    "ongoing": [ /* défis déjà là la période précédente */ ],
    "counts":  { "new": 2, "ongoing": 0, "total": 2 }
  }
}
```

Chaque défi : `id`, `name`, `slug`, `description`, `lore`, `quest_type`, `consumption_type`, `target_value`, `bonus_xp`, `bonus_cashback`, `is_repeatable`, `max_completions`, `is_scheduled`, `coupon` (`{name, amount, percentage}` ou `null`), `badge`, `establishments`.

Règles de sélection :

- **Périodicité appariée** : un rapport `weekly` ne liste que les quêtes `period_type = 'weekly'`, un rapport `monthly` que les `monthly`. Sinon une quête mensuelle apparaîtrait dans le rapport de la semaine où elle démarre, puis à nouveau dans celui du mois.
- **Planning identique à celui du moteur** ([migration 041](../tables/quest_periods.md)) : une quête **sans aucune ligne** `quest_periods` est **permanente** et compte à chaque période ; une quête qui en a ne compte que si la période visée y figure. Annoncer autre chose que ce que le moteur récompense serait pire que de ne rien annoncer.
- **`is_active` obligatoire** : les quêtes archivées ne sortent jamais.
- Est **nouveau** un défi programmé sur cette période et **absent de la précédente** (c'est la rotation qui l'ouvre), ou un défi **permanent créé pendant la période** (pas de planning, mais il n'existait pas avant). Tout le reste est *reconduit*.
- **`max_completions`** = `1 + nombre de lignes quest_iterations` pour une quête répétable, `1` sinon : c'est le **plafond configuré**, que le rang du joueur peut encore réduire ([migration 069](./get_max_quest_completions.md)). À lire comme un maximum, pas comme une promesse.
- **`establishments` vide** = défi global, valable dans toutes les tavernes (aucune ligne `quests_establishments`).
- **Unités** : `bonus_cashback` et `coupon.amount` en centimes (= PdB, 1 PdB = 1 centime) ; `target_value` en centimes **uniquement** pour `quest_type = 'amount_spent'`, en unités directes sinon.

Aucun compteur de participation ici : le rapport annonce ce qui s'ouvre, il ne fait pas le bilan de ce qui a été complété.

## `format_period_label(p_period_type, p_period_identifier)`

Libellé français d'une période, utilisé dans l'objet et le corps de l'e-mail :

| Type | Exemple |
|---|---|
| `monthly` | `juillet 2026` |
| `weekly` (même mois) | `semaine du 14 au 20 juillet 2026` |
| `weekly` (à cheval) | `semaine du 28 juillet au 3 août 2026` |
| `yearly` | `année 2026` |

`STABLE` (et non `IMMUTABLE`) : les bornes de période sont des `timestamptz`, donc sensibles au fuseau de session.

## Voir aussi

- [Tables des rapports e-mail](../tables/email_reports.md)
- [Table `quest_periods`](../tables/quest_periods.md) — la règle de planning que reprend `build_report_new_quests`
- Edge Function `send-email-reports` : [edge-functions/README.md](../edge-functions/README.md)
