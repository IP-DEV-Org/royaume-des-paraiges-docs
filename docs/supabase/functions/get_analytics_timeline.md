# Function: get_analytics_timeline / get_analytics_timeline_global

RPC introduites par la migration **046 (01/06/2026)** pour la refonte de la page `/analytics` (tableau « timeline » par journée fiscale, groupé par établissement). Admin-only.

> Migration **053 (04/06/2026)** : 1ʳᵉ version de la colonne Cashpad (`cashpad_euro_royaume_cents`). **Remplacée par la migration 054** ci-dessous.
>
> Migration **054 (04/06/2026)** : refonte des métriques pour la **réconciliation Cashpad ↔ Royaume** en deux axes (Euros Royaume + Paiements PdB), chacun comparé Cashpad vs Royaume, plus la génération de PdB (organique + quêtes). Retire `pdb_payments_cents` / `transactions_amount_cents` / `cashpad_euro_royaume_cents` au profit de 6 colonnes. `DROP`/`CREATE` : `EXECUTE` ré-octroyé à `authenticated`/`service_role`, révoqué `anon`/`PUBLIC`.
>
> Migration **055 (04/06/2026)** : ajout de `euro_cashpad_other_cents` (paiements Cashpad hors Royaume) → ligne « Euros Cashpad » en tête de tableau. `DROP`/`CREATE`, droits réappliqués à l'identique.
>
> Migration **056 (04/06/2026)** : **comparaison Cashpad rendue optionnelle (perf)**. Nouveau 4ᵉ paramètre `p_include_cashpad boolean DEFAULT false`. L'agrégation des colonnes « selon Cashpad » (`euro_cashpad_*`, `pdb_cashpad_cents`) parcourait tout `cashpad_receipts_snapshot` (~454 MB, JSONB) à chaque appel (~2,4 s). Désormais : **off** (défaut) → CTE `cashpad_pm` court-circuitée (snapshot **jamais lu**), ces 3 colonnes renvoient `NULL` → ~50 ms ; **on** → agrégation réécrite en `CROSS JOIN LATERAL` par clôture (exploite `idx_crs_establishment_closed`, ~0,25 s à chaud). `DROP` de la 3-arg + `CREATE` de la 4-arg, droits réappliqués à l'identique.
>
> Migration **091 (07/09/2026)** : **index manquants** sur trois clés étrangères, sans changement de la RPC. Les deux `LEFT JOIN LATERAL` (lignes de paiement, gains organiques) étaient rejoués en **Seq Scan complet une fois par receipt** : sur une année (6 774 receipts) la RPC mettait **9,4 s**, au-delà du `statement_timeout` de **8 s** du rôle `authenticated` — l'export annuel de `/analytics` échouait en `57014` alors que l'affichage mois par mois passait. Ajout de `idx_receipt_lines_receipt_id`, `idx_gains_receipt_id` et `idx_receipts_customer_created (customer_id, created_at DESC)` (ce dernier pour le sous-select `quest_attr`). Mesure sur 2026, Cashpad off : **9 440 ms → 274 ms**. Cashpad on, un mois : ~4,3 s → ~0,74 s.
>
> Migrations **092 + 093 (07/09/2026)** : **matérialisation des totaux par mode de paiement**. La CTE `cashpad_pm` n'expanse plus le JSONB : elle somme trois colonnes `payments_euro_cents` / `payments_pdb_cents` / `payments_other_cents` posées sur `cashpad_receipts_snapshot` et remplies par le trigger `trg_cashpad_payment_totals` (092), servies par l'index couvrant partiel `idx_crs_payment_totals`. La règle de classement des modes de paiement est déplacée telle quelle dans `public.cashpad_payment_totals(jsonb)` — **équivalence vérifiée sur les 289 457 lignes : 0 divergence**, et empreinte MD5 de la sortie de la RPC identique avant/après sur mai→août 2026. Une **année complète avec comparaison Cashpad passe de ~44 s à ~0,41 s**, ce qui rend l'export annuel possible avec les colonnes Cashpad. Signature inchangée, `CREATE OR REPLACE` (droits conservés).

## Coût de la comparaison Cashpad

Les chiffres de la migration 056 (~0,25 s à chaud) **avaient vieilli avec la table** : `cashpad_receipts_snapshot` pèse 570 Mo / 289 000 lignes, et l'agrégation re-extrayait `payments[]` du JSONB `raw_payload` de chaque ticket **à chaque appel** — 41 s sur une année, donc un export annuel impossible sous le `statement_timeout` de 8 s.

Les migrations **092/093** ont supprimé ce coût en matérialisant les trois totaux sur la ligne du ticket (cf. `tables/cashpad_receipts_snapshot.md`). État actuel :

| Plage | Cashpad off | Cashpad on |
|-------|-------------|------------|
| 1 mois | ~30 ms | ~40 ms |
| 1 an (1 363 clôtures, 287 k tickets) | ~0,3 s | **~0,4 s** |

`p_include_cashpad` reste utile pour ce qu'il **affiche** (les lignes de comparaison), plus pour ce qu'il coûte.

## get_analytics_timeline

### Signature

```sql
CREATE FUNCTION public.get_analytics_timeline(
  p_start_date date,
  p_end_date   date,
  p_establishment_ids int[] DEFAULT NULL,
  p_include_cashpad boolean DEFAULT false  -- off → colonnes Cashpad NULL (rapide)
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

**Colonne « Total » + exports CSV (côté UI admin, 07/2026 — aucun changement RPC).** Le tableau `/analytics` affiche une colonne **Total** figée (somme de chaque ligne de métrique sur les journées affichées ; les lignes *Différence* ne somment que les journées avec donnée Cashpad). Deux exports CSV : **« Exporter CSV »** reprend exactement les lignes/colonnes affichées (filtres établissements + `p_include_cashpad` de l'écran) sur la période courante (Jour/Semaine/Mois, fichier `analytics_<start>_<end>.csv`) ; **« Exporter année »** passe par un appel dédié `get_analytics_timeline(<année>-01-01, <année>-12-31, …)` (fichier `analytics_<année>.csv`), colonnes Cashpad comprises si la comparaison est active. Format : `Établissement;Métrique;Total;<journées>`, séparateur `;`, décimales à virgule, BOM (Excel fr), montants en euros sans symbole.

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
