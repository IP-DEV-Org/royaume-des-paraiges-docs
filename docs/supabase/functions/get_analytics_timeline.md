# Function: get_analytics_timeline / get_analytics_timeline_global

RPC introduites par la migration **046 (01/06/2026)** pour la refonte de la page `/analytics` (tableau « timeline » par journée fiscale, groupé par établissement). Admin-only.

> Migration **053 (04/06/2026)** : 1ʳᵉ version de la colonne Cashpad (`cashpad_euro_royaume_cents`). **Remplacée par la migration 054** ci-dessous.
>
> Migration **054 (04/06/2026)** : refonte des métriques pour la **réconciliation Cashpad ↔ Royaume** en deux axes (Euros Royaume + Paiements PdB), chacun comparé Cashpad vs Royaume, plus la génération de PdB (organique + quêtes). Retire `pdb_payments_cents` / `transactions_amount_cents` / `cashpad_euro_royaume_cents` au profit de 6 colonnes. `DROP`/`CREATE` : `EXECUTE` ré-octroyé à `authenticated`/`service_role`, révoqué `anon`/`PUBLIC`.
>
> Migration **055 (04/06/2026)** : ajout de `euro_cashpad_other_cents` (paiements Cashpad hors Royaume) → ligne « Euros Cashpad » en tête de tableau. `DROP`/`CREATE`, droits réappliqués à l'identique.

## get_analytics_timeline

### Signature

```sql
CREATE FUNCTION public.get_analytics_timeline(
  p_start_date date,
  p_end_date   date,
  p_establishment_ids int[] DEFAULT NULL
) RETURNS TABLE (
  establishment_id     integer,
  establishment_title  text,
  fiscal_date          date,
  range_begin          timestamptz,
  range_end            timestamptz,
  euro_cashpad_other_cents bigint,  -- NULL en fallback (Cashpad hors Royaume)
  euro_cashpad_cents   bigint,   -- NULL en fallback
  euro_royaume_cents   bigint,
  pdb_cashpad_cents    bigint,   -- NULL en fallback
  pdb_royaume_cents    bigint,
  pdb_organic_cents    bigint,
  pdb_quest_cents      bigint,
  is_fallback_calendar boolean
)
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public
```

`EXECUTE` accordé à `authenticated` / `service_role` ; révoqué pour `anon` / `PUBLIC`. Contrôle d'accès via `assert_admin()`.

### Métriques (par établissement et journée fiscale)

L'UI `/analytics` affiche ces colonnes en 3 blocs (séparateurs visuels), tous montants **en euros** (1 PdB = 0,01 €) :

| Colonne | Bloc | Source |
|---|---|---|
| `euro_cashpad_other_cents` | Euros Cashpad | **Paiements Cashpad SANS lien Royaume** : `SUM(payments[].amount)` des modes ≠ `%royaume%` ET ≠ `%paraige%` (Euros simples, CB, Espèces, Ticket resto, Virement…), millièmes ÷10. `NULL` en fallback. Ligne « Euros Cashpad » affichée **tout en haut** (sans comparaison). Additif : `euro_cashpad_other_cents + euro_cashpad_cents + pdb_cashpad_cents` = total des paiements Cashpad de la clôture. |
| `euro_cashpad_cents` | Euros Royaume | Mode de paiement Cashpad **« Euros Royaume »** : `SUM(payments[].amount)` (filtre `lower(name) LIKE '%royaume%'`) sur les tickets non annulés de la clôture (`closed_at ∈ [range_begin, range_end]`), **millièmes ÷10**. `NULL` en fallback. |
| `euro_royaume_cents` | Euros Royaume | `SUM(receipt_lines.amount)` où `payment_method IN ('card','cash')` — euros des receipts scannés (hors PdB). |
| `pdb_cashpad_cents` | Paiements PdB | Mode de paiement Cashpad **« Paraiges de Bronze »** (filtre `LIKE '%paraige%'`), même logique que `euro_cashpad_cents`. `NULL` en fallback. ⚠️ Mode très peu utilisé en caisse → souvent 0. |
| `pdb_royaume_cents` | Paiements PdB | `SUM(receipt_lines.amount)` où `payment_method = 'cashback'` — PdB réglés sur les receipts scannés. |
| `pdb_organic_cents` | Génération | `SUM(gains.cashback_money)` où `source_type = 'receipt'` (cashback organique). |
| `pdb_quest_cents` | Génération | `SUM(gains.cashback_money)` où `source_type = 'bonus_cashback_quest'`. Ces gains n'ont **ni `receipt_id` ni `establishment_id`** → rattachés par **heuristique** à l'établissement du **dernier receipt** du client dont `created_at ≤ gain.created_at` (le gain est créé dans la foulée du receipt déclencheur). |

**Comparaison Cashpad ↔ Royaume.** L'UI calcule deux lignes **Différence (Cashpad − Royaume)** : `euro_cashpad_cents − euro_royaume_cents` et `pdb_cashpad_cents − pdb_royaume_cents`. Vert si écart nul, ambre sinon (divergence à investiguer : paiement saisi en caisse sans scan QR correspondant, ou inversement). ⚠️ Les agrégats Cashpad sont **par clôture** (pas par receipt) ; ils n'apparaissent que si l'établissement a ≥ 1 receipt Royaume sur la journée fiscale (sinon aucune ligne dans le tableau).

> Les gains **hors quête sans établissement** (`bonus_cashback_leaderboard`, `bonus_cashback_manual`, `rollback_beta_correction`) ne sont **pas** affichés sur `/analytics` (décision produit 04/06/2026). Le bloc global « toutes enseignes » (et donc l'usage de `get_analytics_timeline_global` côté page) a été retiré ; la RPC `_global` reste en base mais n'est plus appelée.

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
