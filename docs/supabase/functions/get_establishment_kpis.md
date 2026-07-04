# Function: get_establishment_kpis

KPIs par établissement pour la page admin `/analytics/establishments` (migration **062**, juillet 2026). Une ligne par établissement — les établissements sans activité sur la période sortent avec des zéros (LEFT JOIN depuis `establishments`).

## Signature

```sql
CREATE FUNCTION get_establishment_kpis(
  p_start_date DATE,
  p_end_date   DATE
) RETURNS TABLE (
  establishment_id    INTEGER,
  establishment_title TEXT,
  sales_count         BIGINT,
  euro_cents          BIGINT,
  pdb_spent_cents     BIGINT,
  pdb_generated_cents BIGINT,
  new_clients         BIGINT,
  active_clients      BIGINT,
  employees_count     BIGINT
)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public'
```

## Parametres

| Parametre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `p_start_date` | `DATE` | Oui | Premier jour de la période (inclus) |
| `p_end_date` | `DATE` | Oui | Dernier jour de la période (**inclus**) |

Les bornes sont des **jours calendaires Europe/Paris inclusifs** : la fenêtre effective est `[p_start_date 00:00 Paris, (p_end_date + 1) 00:00 Paris)`. Mêmes conventions que `getPeriodBounds` côté admin.

## Retour

| Champ | Description |
|-------|-------------|
| `sales_count` | Nombre de `receipts` sur la période |
| `euro_cents` | CA euros = `receipt_lines` `card` + `cash` (centimes) |
| `pdb_spent_cents` | Paiements PdB = `receipt_lines` `cashback` (centimes, 1 PdB = 1 centime) |
| `pdb_generated_cents` | PdB organiques générés = `gains` `source_type='receipt'` (centimes) |
| `new_clients` | Clients dont le **premier receipt all-time dans CET établissement** tombe dans la période — un client déjà connu ailleurs mais nouveau ici compte comme nouveau ici. Un appel sur la période précédente donne naturellement le comparatif. |
| `active_clients` | Clients distincts ayant ≥ 1 receipt sur la période |
| `employees_count` | Effectif **actuel** (indépendant de la période) : `profiles` `role IN ('employee','establishment')` avec `attached_establishment_id`, `deleted_at IS NULL`, `NOT is_test` |

## Exclusions

- Comptes test (`profiles.is_test = true`) et compte système `cashpad-system@royaume.internal` exclus de toutes les métriques d'activité (mêmes règles que `get_analytics_timeline`).
- Pas de filtre `deleted_at` sur les métriques receipts/gains (un client anonymisé RGPD a quand même généré du CA — cohérent avec les autres RPC analytics) ; `deleted_at IS NULL` uniquement pour `employees_count`.

## Securite

- `PERFORM assert_admin()` — **admin only** (contrairement aux RPC `get_analytics_revenue`/`_debts` qui acceptent les rôles establishment/employee scopés).
- `REVOKE FROM PUBLIC, anon` ; `GRANT EXECUTE TO authenticated, service_role`.

## Exemple

```typescript
// Service admin : src/lib/services/analyticsService.ts → getEstablishmentKpis
const { data } = await (supabase.rpc as any)('get_establishment_kpis', {
  p_start_date: '2026-06-01',
  p_end_date: '2026-06-30',
});
```

La page `/analytics/establishments` l'appelle **deux fois** (période courante + période calendaire précédente via `shiftPeriod`) pour calculer les évolutions en % (nouveaux clients, CA, tickets…).
