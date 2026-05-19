# Function: get_analytics_debts

Retourne le breakdown des dettes en Paraiges de Bronze par categorie, avec filtres optionnels.

## Signature

```sql
CREATE FUNCTION get_analytics_debts(
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
| `p_establishment_id` | `BIGINT` | Non | Filtre par etablissement (uniquement PdB organiques) |
| `p_employee_id` | `UUID` | Non | Filtre par employe (uniquement PdB organiques, via receipts) |

## Retour

```json
{
  "pdb_organic": 762,
  "pdb_rewards": 2780,
  "pdb_bonus_coupons": 1000,
  "pdb_total": 4542,
  "active_pct_coupons_count": 3,
  "has_filter": false
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `pdb_organic` | INTEGER | PdB gagnes via depenses euros (source_type='receipt') |
| `pdb_rewards` | INTEGER | PdB gagnes via quetes/classement |
| `pdb_bonus_coupons` | INTEGER | PdB gagnes via coupons manuels/triggers/migration |
| `pdb_total` | INTEGER | Total des 3 categories |
| `active_pct_coupons_count` | INTEGER | Nombre de coupons % actifs non utilises |
| `has_filter` | BOOLEAN | True si un filtre etablissement ou employe est actif |

## Categories par source_type

| Categorie | source_type values |
|-----------|-------------------|
| Organique | `receipt` |
| Recompenses | `bonus_cashback_quest`, `bonus_cashback_leaderboard` |
| Bonus Coupons | `bonus_cashback_manual`, `bonus_cashback_trigger`, `bonus_cashback_migration` |

> **Note** : La categorie "Bonus Coupons" est toujours calculee par le backend (champ `pdb_bonus_coupons` dans la reponse), mais n'est plus affichee dans le frontend admin depuis fevrier 2026. Les donnees existantes ont ete reclassees en `bonus_cashback_leaderboard`. Le champ reste disponible pour un usage futur si necessaire.

## Comportement avec filtres

Quand `p_establishment_id` ou `p_employee_id` est specifie :
- **PdB Organiques** : filtres normalement (via `gains.establishment_id` ou JOIN `receipts.employee_id`)
- **PdB Recompenses** : retourne 0 (pas de lien etablissement/employe)
- **PdB Bonus Coupons** : retourne 0 (pas de lien etablissement/employe)
- **Coupons % actifs** : toujours global (pas filtre)

## Securite

Depuis la migration **040 (Security Definer hardening, 18/05/2026)** :

- Guard `assert_admin_or_establishment_for(p_establishment_id)` (voir [`assert_admin_or_establishment_for.md`](./assert_admin_or_establishment_for.md))
- **Anti-spoof** : pour un appelant `role IN ('establishment', 'employee')`, `p_establishment_id` est silencieusement override par `profiles.attached_establishment_id` avant le guard. Seul un admin peut lire les données globales (`p_establishment_id = NULL`) ou d'un autre etab.
- Bypass automatique pour `service_role` et superuser.
- Exclusion `NOT p.is_test` appliquée sur toutes les agrégations (organic, rewards, bonus_coupons, coupons %).

## Exemple

```typescript
const { data } = await (supabase.rpc as any)('get_analytics_debts', {
  p_start_date: '2026-02-01T00:00:00Z',
  p_end_date: '2026-02-18T00:00:00Z',
});
```
