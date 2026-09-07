# Table: cashpad_receipts_snapshot

## Description

Cache local des tickets Cashpad récupérés via la Sales Retrieval API (`salesdata/v2`) par l'edge function `cashpad-reconcile-daily`. Sert de contrepartie aux receipts Royaume dans la [réconciliation](../functions/get_analytics_timeline.md) et de source des colonnes « selon Cashpad » de `/analytics`.

Introduite par les migrations **032** et **035**.

Contient **tous** les tickets remontés, y compris les `cancelled = true` et les tickets à 0 € : ils sont exclus du matching mais conservés pour l'audit et pour le signal « scan client sur ticket annulé » (`cashpad_reconciliations.cancelled_match_id`).

## Grain

Une ligne par ticket Cashpad, PK `cashpad_receipt_id` (texte, identifiant Cashpad). ~289 000 lignes / 570 Mo à date (septembre 2026), dont l'essentiel est le JSONB `raw_payload` (~1,7 Ko par ligne, stocké **en ligne** et non déporté en TOAST).

## Schema

| Colonne | Type | Nullable | Description |
|---------|------|----------|-------------|
| `cashpad_receipt_id` | text | Non | PK. Identifiant du ticket côté Cashpad. |
| `cashpad_sequential_id` | integer | Oui | Numéro séquentiel du ticket. |
| `installation_id` | text | Non | Installation Cashpad d'origine (→ mapping vers `establishment_id`). |
| `establishment_id` | integer | Oui | Établissement Royaume résolu depuis l'installation. |
| `amount_cents` | integer | Non | Montant du ticket en centimes (millièmes Cashpad ÷ 10). |
| `closed_at` | timestamptz | Non | Horodatage de clôture du ticket. |
| `cashpad_user_id` / `cashpad_user_name` | text | Oui | Serveur Cashpad (`owner`), utilisé pour départager les matchs ambigus. |
| `products` | jsonb | Oui | Extrait de `items[]`. |
| `raw_payload` | jsonb | Non | Payload Cashpad complet. Source de vérité, jamais réécrit partiellement. |
| `fetched_at` | timestamptz | Non | Date de récupération. |
| `cancelled` | boolean | Non | Ticket annulé côté caisse. |
| `payments_euro_cents` | bigint | Oui | **Migration 092.** Montant encaissé via le mode de paiement Cashpad « Euros Royaume », en centimes. |
| `payments_pdb_cents` | bigint | Oui | **Migration 092.** Montant encaissé via « Paraiges de Bronze », en centimes. |
| `payments_other_cents` | bigint | Oui | **Migration 092.** Tout autre mode de paiement (CB, espèces, ticket resto…), c.-à-d. les encaissements **sans lien Royaume**, en centimes. |

## Totaux par mode de paiement (migration 092)

### Pourquoi

Ces trois montants étaient re-extraits du JSONB `raw_payload` **à chaque appel** de `get_analytics_timeline(p_include_cashpad => true)`. Sur une année cela représente 287 000 tickets à parcourir et décompresser, soit **~41 s** — au-delà du `statement_timeout` de 8 s du rôle `authenticated`, donc un export annuel impossible.

Or ces valeurs sont **figées** : une fois un service clôturé, les paiements d'un ticket ne changent plus. On recalculait en permanence une donnée immuable. Elles sont désormais stockées sur la ligne.

### Comment

- **`public.cashpad_payment_totals(jsonb)`** (`IMMUTABLE`, `PARALLEL SAFE`) porte la **règle de classement**, seule source de vérité : `LIKE '%royaume%'` → euro, `LIKE '%paraige%'` → PdB, le reste → other ; montants Cashpad en millièmes ÷ 10.
- **`trg_cashpad_payment_totals`** (`BEFORE INSERT OR UPDATE`) remplit les trois colonnes. Il couvre **tous** les chemins d'écriture, y compris le ré-upsert d'un ticket par `cashpad-reconcile-daily` : aucune dérive possible, la valeur est toujours recalculée depuis le `raw_payload` courant. Sur `UPDATE`, l'extraction est court-circuitée si `raw_payload` n'a pas changé.
- **`idx_crs_payment_totals`** : `(establishment_id, closed_at) INCLUDE (payments_*) WHERE cancelled = false` → parcours d'index seul, le heap (et donc `raw_payload`) n'est jamais touché par l'agrégation.

Résultat : agrégation d'une année **41,4 s → 0,41 s**.

### Pièges

- ⚠️ Les colonnes sont **nullables**, mais aucune ligne ne devrait avoir de `NULL` : `NULL` signifie « pas encore backfillée ». Un `NULL` qui apparaîtrait signalerait une écriture ayant contourné le trigger (`COPY ... FREEZE`, désactivation de trigger, restauration partielle).
- Un **backfill** de cette table réécrit ~484 Mo de heap (le payload tient en ligne) : toujours procéder **par lots**, jamais en une transaction unique. La production a été backfillée ainsi le 07/09/2026, avec un index partiel temporaire sur les lignes `NULL` pour éviter un Seq Scan quadratique, puis `VACUUM ANALYZE`.
- Modifier la règle de classement des modes de paiement impose de **rejouer le backfill** : changer `cashpad_payment_totals` n'a aucun effet rétroactif sur les lignes déjà calculées.

## Index

| Index | Définition | Usage |
|-------|-----------|-------|
| `cashpad_receipts_snapshot_pkey` | `(cashpad_receipt_id)` | PK, upsert de l'edge function. |
| `idx_crs_establishment_closed` | `(establishment_id, closed_at)` | Sélection des tickets d'une clôture. |
| `idx_crs_amount` | `(establishment_id, amount_cents, closed_at)` | Recherche de contrepartie par montant strict. |
| `idx_crs_not_cancelled` | idem, `WHERE cancelled = false` | Matching sur les seuls tickets valides. |
| `idx_crs_payment_totals` | `(establishment_id, closed_at) INCLUDE (payments_euro_cents, payments_pdb_cents, payments_other_cents) WHERE cancelled = false` | **Migration 093.** Agrégation Cashpad de `/analytics` en index-only scan. |
