# Table: establishment_consumption_types

Types de consommation proposés par chaque établissement. Permet de filtrer les types affichés dans le sélecteur de consommation du scanner et de l'app serveurs.

**Aucune entrée pour un établissement = tous les types actifs** (comportement par défaut).

## Informations

| Propriete | Valeur |
|-----------|--------|
| **Schema** | `public` |
| **RLS** | Active |

## Colonnes

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | `bigint` | Non | - | Identifiant unique |
| `establishment_id` | `integer` | Non | - | Établissement (FK → establishments) |
| `consumption_type` | `consumption_type` | Non | - | Type de consommation (enum) |
| `is_active` | `boolean` | Non | true | Actif ou non |
| `created_at` | `timestamp with time zone` | Non | now() | Date de création |

## Cles primaires

- `id`

## Contraintes

| Contrainte | Type | Colonne(s) |
|-----------|------|------------|
| `establishment_consumption_typ_establishment_id_consumption__key` | UNIQUE | `(establishment_id, consumption_type)` |

## Relations (Foreign Keys)

| Colonne | Table cible | Colonne cible |
|---------|------------|---------------|
| `establishment_id` | `establishments` | `id` |

## Politiques RLS

| Politique | Commande | Description |
|-----------|----------|-------------|
| `Authenticated users can read establishment consumption types` | SELECT | Tous les utilisateurs authentifiés |
| `Admins can manage establishment consumption types` | ALL | Role admin uniquement |

## Valeurs possibles (enum `consumption_type`)

`biere`, `cocktail`, `alcool`, `soft`, `boisson_chaude`, `restauration`
