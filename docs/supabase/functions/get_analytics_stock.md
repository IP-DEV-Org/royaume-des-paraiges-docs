# Function: get_analytics_stock

Retourne le stock de Paraiges de Bronze (ouverture et fermeture) par categorie pour une periode donnee, avec allocation proportionnelle des depenses.

## Signature

```sql
CREATE FUNCTION get_analytics_stock(
  p_start_date TIMESTAMPTZ,
  p_end_date TIMESTAMPTZ,
  p_establishment_id BIGINT DEFAULT NULL
) RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public'
```

## Parametres

| Parametre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `p_start_date` | `TIMESTAMPTZ` | Oui | Debut de la periode |
| `p_end_date` | `TIMESTAMPTZ` | Oui | Fin de la periode |
| `p_establishment_id` | `BIGINT` | Non | Filtre par etablissement (seuls les PdB organiques sont filtres) |

## Retour

```json
{
  "opening": {
    "organic": 500,
    "rewards": 300,
    "bonus_coupons": 200,
    "total_earned": 1000,
    "total_spent": 400,
    "total": 600
  },
  "closing": {
    "organic": 650,
    "rewards": 400,
    "bonus_coupons": 250,
    "total_earned": 1300,
    "total_spent": 500,
    "total": 800
  },
  "movements": {
    "earned_organic": 150,
    "earned_rewards": 100,
    "earned_bonus_coupons": 50,
    "spent": 100
  },
  "has_filter": false
}
```

## Logique de calcul

### Stock d'ouverture
- Somme des gains par categorie ou `created_at < p_start_date`
- Somme des spendings ou `created_at < p_start_date`

### Stock de fermeture
- Somme des gains par categorie ou `created_at < p_end_date`
- Somme des spendings ou `created_at < p_end_date`

### Allocation proportionnelle
La table `spendings` ne distingue pas quelle categorie de PdB est depensee. Les depenses sont reparties proportionnellement :

```
spent_category = total_spent * (earned_category / total_earned)
stock_category = earned_category - spent_category
```

Avec `GREATEST(..., 0)` pour eviter les valeurs negatives dues aux arrondis.

### Mouvements
Difference entre les totaux de fermeture et d'ouverture pour chaque categorie.

> **Note** : La categorie "Bonus Coupons" (`bonus_coupons`, `earned_bonus_coupons`) est toujours calculee par le backend, mais n'est plus affichee dans le frontend admin depuis fevrier 2026. Les donnees existantes ont ete reclassees en `bonus_cashback_leaderboard`. Les champs restent disponibles pour un usage futur si necessaire.

## Comportement avec filtre etablissement

Quand `p_establishment_id` est specifie :
- PdB Organiques : filtres via `gains.establishment_id`
- PdB Recompenses et Bonus Coupons : retournent 0 (pas de lien etablissement)
- Spendings : filtres via `spendings.establishment_id`

## Securite

Depuis la migration **040 (Security Definer hardening, 18/05/2026)** :

- Guard `assert_admin_or_establishment_for(p_establishment_id)` (voir [`assert_admin_or_establishment_for.md`](./assert_admin_or_establishment_for.md))
- **Anti-spoof** : pour un appelant `role IN ('establishment', 'employee')`, `p_establishment_id` est silencieusement override par `profiles.attached_establishment_id` avant le guard. Seul un admin peut lire les données globales ou d'un autre etab.
- Bypass automatique pour `service_role` et superuser.
- Exclusion `NOT p.is_test` appliquée sur toutes les agrégations (gains et spendings, ouverture et fermeture).
- Cette fonction ne prend pas de `p_employee_id` (le stock n'est pas attribuable à un employé).

## Exemple

```typescript
const { data } = await (supabase.rpc as any)('get_analytics_stock', {
  p_start_date: '2026-02-01T00:00:00Z',
  p_end_date: '2026-02-18T00:00:00Z',
});
```
