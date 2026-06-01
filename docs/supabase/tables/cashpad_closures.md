# Table: cashpad_closures

## Description

Fenêtres de **journée fiscale** (clôture → clôture) persistées par établissement. La réconciliation Cashpad raisonne par clôture de caisse et non de minuit à minuit ; les bornes `[range_begin, range_end]` d'une clôture étaient jusqu'ici calculées à la volée par l'edge function `cashpad-reconcile-daily` (via `getArchivesOpenedOnDate`) puis jetées. Cette table les **persiste** pour alimenter la page analytics « timeline » (`/analytics`), qui rattache chaque receipt Royaume à la bonne journée fiscale.

Introduite par la migration **045 (01/06/2026)**.

## Grain

Une ligne par `(establishment_id, fiscal_date)`. Un groupe d'établissements partageant une même installation Cashpad produit **N lignes identiques** (une par membre) : la fenêtre est répliquée sur chaque membre pour que les requêtes analytics joignent purement sur `establishment_id`, sans résolution de groupe.

`fiscal_date` = jour calendaire (UTC) d'**OUVERTURE** du service (`range_begin_date::date`). Une clôture faite le 31 à 02h pour un service ouvert le 30 au soir est rattachée au **30**. L'ouverture se faisant en soirée, la date UTC == date locale.

## Schema

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | bigint (identity) | Non | - | PK. |
| `establishment_id` | integer | Non | - | FK `establishments(id)` ON DELETE CASCADE. |
| `fiscal_date` | date | Non | - | Jour d'ouverture du service (= libellé de colonne dans la timeline). |
| `range_begin` | timestamptz | Non | - | `min(range_begin_date)` des archives du jour. NON paddé. |
| `range_end` | timestamptz | Non | - | `max(range_end_date)` des archives du jour. NON paddé. |
| `installation_id` | text | Non | - | Installation Cashpad ayant produit la clôture. |
| `cashpad_sequential_id` | integer | Oui | - | `sequential_id` max des archives fusionnées (audit). |
| `archive_count` | integer | Non | 1 | Nombre d'archives (services) fusionnées dans la fenêtre. |
| `updated_at` | timestamptz | Non | now() | Horodatage du dernier upsert. |

## Contraintes

- `UNIQUE (establishment_id, fiscal_date)` → idempotence des upserts de l'edge function.
- `CHECK (range_end >= range_begin)`.
- Index `(establishment_id, range_begin, range_end)` (bucketing des receipts) et `(establishment_id, fiscal_date)` (dérivation des colonnes).

## RLS

RLS active : **Oui**

| Policy | Action | Condition |
|---|---|---|
| `cashpad_closures_admin_select` | SELECT | `profiles.role = 'admin'` |

Pas de policy INSERT/UPDATE : l'écriture passe exclusivement par le client `service_role` de l'edge function `cashpad-reconcile-daily` (bypass RLS), comme `cashpad_receipts_snapshot`.

## Alimentation

Peuplée par `cashpad-reconcile-daily` (`upsertClosures`) juste après l'upsert des snapshots, pour chaque établissement membre de la cible, dès qu'un service a ouvert ce jour-là. Le **backfill** historique se fait en relançant la réconciliation en mode global (idempotent via l'unique key).

## Consommateurs

- RPC `get_analytics_timeline` (page `/analytics`) : rattachement fiscal des receipts.
