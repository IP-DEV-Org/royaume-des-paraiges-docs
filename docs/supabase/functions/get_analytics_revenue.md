# Function: get_analytics_revenue

Retourne le breakdown des recettes (paiements euros et PdB depenses) pour une periode donnee, avec filtres optionnels.

## Signature

```sql
CREATE FUNCTION get_analytics_revenue(
  p_start_date TIMESTAMPTZ,
  p_end_date TIMESTAMPTZ,
  p_establishment_id BIGINT DEFAULT NULL,
  p_employee_id UUID DEFAULT NULL
) RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public'
```

## Parametres

| Parametre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `p_start_date` | `TIMESTAMPTZ` | Oui | Debut de la periode (inclus) |
| `p_end_date` | `TIMESTAMPTZ` | Oui | Fin de la periode (exclu) |
| `p_establishment_id` | `BIGINT` | Non | Filtre par etablissement |
| `p_employee_id` | `UUID` | Non | Filtre par employe (via receipts.employee_id) |

## Retour

```json
{
  "sales_count": 42,
  "total_euros": 125000,
  "card_total": 100000,
  "cash_total": 20000,
  "cashback_spent_total": 5000
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `sales_count` | INTEGER | Nombre de receipts sur la periode |
| `total_euros` | INTEGER | Total carte + especes (centimes) |
| `card_total` | INTEGER | Total paiements carte (centimes) |
| `cash_total` | INTEGER | Total paiements especes (centimes) |
| `cashback_spent_total` | INTEGER | Total PdB depenses par les clients (centimes) |

## Logique

1. Lit le `role` et `attached_establishment_id` de l'appelant depuis `profiles` via `auth.uid()`
2. **Anti-spoof** : si role ∈ `{establishment, employee}`, override `p_establishment_id := attached_establishment_id` (peu importe ce que le client a envoyé)
3. `PERFORM public.assert_admin_or_establishment_for(p_establishment_id)` — bloque si l'appelant n'est ni admin ni rattaché à cet etab (voir Securite)
4. Compte les receipts correspondant aux filtres (periode + etablissement + employe), avec exclusion `NOT p.is_test`
5. Joint `receipt_lines` aux receipts filtres
6. Agrege les montants par `payment_method` (card, cash, cashback)

## Securite

Depuis la migration **040 (Security Definer hardening, 18/05/2026)** :

- Guard `assert_admin_or_establishment_for(p_establishment_id)` (voir [`assert_admin_or_establishment_for.md`](./assert_admin_or_establishment_for.md))
- Pour un appelant `role IN ('establishment', 'employee')`, le `p_establishment_id` est **silencieusement écrasé** par `profiles.attached_establishment_id` avant le guard — impossible de lire les analytics d'un autre etab via le paramètre.
- Seul un admin peut filtrer librement (n'importe quel `p_establishment_id` ou `NULL` pour la vue globale).
- Bypass automatique pour `service_role` et superuser.
- Exclusion `NOT p.is_test` appliquée systématiquement (receipts ET receipt_lines).

## Exemple

```typescript
const { data } = await (supabase.rpc as any)('get_analytics_revenue', {
  p_start_date: '2026-02-01T00:00:00Z',
  p_end_date: '2026-02-18T00:00:00Z',
  p_establishment_id: 1,
});
```
