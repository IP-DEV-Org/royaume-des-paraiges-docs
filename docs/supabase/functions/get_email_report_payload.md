# Fonction : `get_email_report_payload`

Point d'entrée unique des données des **rapports e-mail** (migration 077, août 2026). Résout la période visée, dispatche vers le builder correspondant au `report_type` et renvoie un payload prêt à rendre. Toute la logique métier des rapports vit ici, en SQL : l'Edge Function `send-email-reports` ne fait que du rendu HTML et de l'envoi.

```sql
get_email_report_payload(
  p_report_key        TEXT,
  p_period_identifier TEXT DEFAULT NULL
) RETURNS JSONB
```

- `SECURITY DEFINER` + `assert_admin()` → accessible aux **admins** et au **service_role** (donc à l'Edge Function et au cron).
- Sans `p_period_identifier`, la période visée est la **période écoulée** (`get_previous_period_identifier`). Un rapport sur la période en cours donnerait des chiffres partiels.
- Erreur `P0425` (`EMAIL_REPORT_UNKNOWN`) si la clé de rapport n'existe pas, `EMAIL_REPORT_TYPE_UNSUPPORTED` si le type n'a pas de builder.
- `EXECUTE` révoqué à `PUBLIC` et `anon`, accordé à `authenticated` et `service_role`.

## Forme du retour

```jsonc
{
  "report": { "key", "name", "report_type", "period_type", "subject" },
  "period": { "identifier", "label", "start", "end" },
  "data":   { /* dépend du report_type, voir builders */ },
  "generated_at": "2026-08-20T08:34:42Z"
}
```

`report.subject` est déjà interpolé : `{{period_label}}` du `subject_template` y est remplacé par le libellé humain.

## Builders

Les deux builders sont `SECURITY DEFINER` et **révoqués à `PUBLIC`, `anon` et `authenticated`** : ils ne sont appelables que depuis le dispatcher.

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
- Edge Function `send-email-reports` : [edge-functions/README.md](../edge-functions/README.md)
