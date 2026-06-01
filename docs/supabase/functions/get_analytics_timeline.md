# Function: get_analytics_timeline / get_analytics_timeline_global

RPC introduites par la migration **046 (01/06/2026)** pour la refonte de la page `/analytics` (tableau « timeline » par journée fiscale, groupé par établissement). Admin-only.

## get_analytics_timeline

### Signature

```sql
CREATE FUNCTION public.get_analytics_timeline(
  p_start_date date,
  p_end_date   date,
  p_establishment_ids int[] DEFAULT NULL
) RETURNS TABLE (
  establishment_id          integer,
  establishment_title       text,
  fiscal_date               date,
  range_begin               timestamptz,
  range_end                 timestamptz,
  pdb_payments_cents        bigint,
  transactions_amount_cents bigint,
  pdb_organic_cents         bigint,
  is_fallback_calendar      boolean
)
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public
```

`EXECUTE` accordé à `authenticated` / `service_role` ; révoqué pour `anon` / `PUBLIC`. Contrôle d'accès via `assert_admin()`.

### Métriques (par établissement et journée fiscale)

| Colonne | Source |
|---|---|
| `pdb_payments_cents` | `SUM(receipt_lines.amount)` où `payment_method = 'cashback'` (paiements réglés en PdB). |
| `transactions_amount_cents` | `SUM(receipts.amount)` (montant total des tickets enregistrés sur le Royaume). |
| `pdb_organic_cents` | `SUM(gains.cashback_money)` où `source_type = 'receipt'` (PdB organiques générés). |

Toutes les valeurs `_cents` sont en **centimes** : l'UI les affiche en **euros** (virgule décimale). 1 PdB = 1 centime, donc les montants PdB sont valorisés en €.

### Rattachement à la journée fiscale

Chaque receipt non-test / non-système est associé à la clôture (`cashpad_closures`) de son établissement dont la fenêtre `[range_begin, range_end]` (paddée ±300 s) contient son `created_at`. Départage (`DISTINCT ON`) : containment strict (non paddé) prioritaire, sinon fenêtre dont le centre est le plus proche.

**Fallback** (`is_fallback_calendar = true`) : si aucune clôture ne couvre le receipt (établissement sans Cashpad, ou période pas encore backfillée), il est bucketé par **jour calendaire `Europe/Paris`**. La colonne correspondante est badgée « cal. » dans l'UI.

### Exclusions

Profils `is_test = true` et compte `cashpad-system@royaume.internal`.

## get_analytics_timeline_global

### Signature

```sql
CREATE FUNCTION public.get_analytics_timeline_global(
  p_start_date date,
  p_end_date   date
) RETURNS TABLE (
  fiscal_date               date,
  pdb_reward_cents          bigint,
  pdb_total_generated_cents bigint
)
```

| Colonne | Source |
|---|---|
| `pdb_reward_cents` | `SUM(gains.cashback_money)` où `source_type <> 'receipt'` (récompense). |
| `pdb_total_generated_cents` | `SUM(gains.cashback_money)` (organique + récompense). |

Bucketé par **jour calendaire `Europe/Paris`** : les PdB récompense (`bonus_cashback_*`) n'ont ni `establishment_id` ni clôture rattachée. Affichage **provisoire** (bloc global « Royaume — toutes enseignes ») en attendant le futur **modèle de dettes inter-établissements** qui les ventilera au prorata des dépenses qualifiantes. Cf. [`design/dettes-inter-etablissements.md`](../design/dettes-inter-etablissements.md).

## Source des bornes fiscales

Voir [table `cashpad_closures`](../tables/cashpad_closures.md), peuplée par l'edge function `cashpad-reconcile-daily`.
