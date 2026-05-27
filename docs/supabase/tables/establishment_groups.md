# Table: establishment_groups

Regroupe plusieurs établissements Royaume partageant une même réalité métier (caisse Cashpad commune, comptoirs adjacents…). Les calculs de visite distincte traitent les établissements d'un même groupe comme un seul.

## Informations

| Propriete | Valeur |
|-----------|--------|
| **Schema** | `public` |
| **RLS** | Active |

## Colonnes

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | `integer` | Non | auto-increment | Identifiant unique |
| `slug` | `text` | Non | - | Identifiant textuel unique |
| `name` | `text` | Non | - | Nom du groupe |
| `cashpad_installation_id` | `text` | Oui | - | ID installation Cashpad partagée (UNIQUE) |
| `created_at` | `timestamp with time zone` | Non | now() | Date de création |

## Cles primaires

- `id`

## Contraintes

| Contrainte | Type | Colonne(s) |
|-----------|------|------------|
| `establishment_groups_slug_key` | UNIQUE | `slug` |
| `establishment_groups_cashpad_installation_id_key` | UNIQUE | `cashpad_installation_id` |

## Politiques RLS

| Politique | Commande | Description |
|-----------|----------|-------------|
| `establishment_groups_admin_read` | SELECT | Role admin uniquement |
| `establishment_groups_admin_write` | ALL | Role admin uniquement |

## Utilisation

Les établissements d'un même groupe sont traités comme un seul pour :
- Le calcul des visites distinctes (badges `establishments_visited`, `all_establishments_visited`)
- Le matching Cashpad (`compute_cashpad_matching_params` est group-aware)

La colonne `group_id` dans `establishments` fait référence à cette table.
