# Table: level_thresholds

## Description

Grille des niveaux du Royaume (refondue avril 2026, migration `005_levels_25_grid.sql`).

⚠️ La grille compte **26 lignes** depuis l'ajout du palier 26 « Chevalier de la table Ronde » via l'UI `/content/storytelling` — le nom de la migration et le chiffre 25 sont donc trompeurs. La table est éditable en admin : **toujours lire le nombre de paliers dynamiquement**, jamais le supposer.

Progression narrative ancrée dans le récit fondateur (Quête de la Pinte Éternelle) :

| Plage niveau | Rang | Coefficient PdB |
|---|---|---:|
| 1 → 5 | Écuyer I → V | 1,0 → 1,8 |
| 6 → 10 | Soldat I → V | 2,0 → 2,8 |
| 11 → 15 | Sergent I → V | 3,0 → 3,8 |
| 16 → 20 | Capitaine I → V | 4,0 → 4,8 |
| 21 → 25 | Chevalier I → V | 5,0 → 5,8 |
| 26 | Chevalier de la table Ronde | 6,0 |

Le niveau d'un Compagnon est dérivé dynamiquement de son XP de la **saison courante** (année calendaire). À chaque variation de gains, un trigger recalcule `profiles.cashback_coefficient = 100 + (level - 1) * 20`.

## Schema

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | integer | Non | - | ID (= `level`, mapping naturel) |
| `level` | integer | Non | - | Numéro du niveau (1 → 26 à date) |
| `xp_required` | integer | Non | - | XP cumulé requis pour atteindre ce niveau |
| `name` | varchar | Non | - | Nom complet du palier (ex. « Sergent III ») |
| `description` | text | Oui | - | Description fonctionnelle courte |
| `lore` | text | Oui | - | Fragment narratif ~150 caractères affiché côté client |
| `sort_order` | integer | Oui | 0 | Ordre de tri (= `level`) |
| `created_at` | timestamptz | Oui | now() | Date de création |
| `updated_at` | timestamptz | Oui | now() | Date de dernière modification (trigger) |

## Clé primaire

- `id`

## RLS

RLS active : **Oui** (lecture publique)

## Helpers PostgreSQL associés

Définis dans `006_cashback_coefficient_trigger.sql` :

- `compute_level_from_xp(p_xp integer) → integer` : niveau correspondant à un XP donné
- `get_season_xp(p_customer_id uuid) → integer` : total XP du Compagnon sur la saison courante
- `update_cashback_coefficient(p_customer_id uuid)` : recalcule `profiles.cashback_coefficient`
- Trigger `gains_refresh_cashback_coefficient` : déclenché AFTER INSERT/UPDATE/DELETE sur `gains`

## Exemples de requêtes

```sql
-- Tous les niveaux ordonnés
SELECT * FROM level_thresholds ORDER BY level ASC;

-- Niveau atteint avec X XP
SELECT compute_level_from_xp(2500); -- → 14 (Sergent IV)

-- XP saison courante d'un client
SELECT get_season_xp('uuid-customer'::uuid);

-- Coefficient courant d'un client (en lisible)
SELECT (cashback_coefficient::float / 100) AS coef
FROM profiles WHERE id = 'uuid-customer'::uuid;
```

## Triggers

| Trigger | Event | Description |
|---------|-------|-------------|
| `set_level_thresholds_updated_at` | BEFORE UPDATE | Met à jour `updated_at` automatiquement |

## Lien avec la saison

Au reset 31/12 (`reset_season(year)`), `cashback_coefficient` redescend à 100 pour tout le monde. Le niveau « repart à 1 » implicitement car `get_season_xp` filtre par année courante.

## Statistiques

- Lignes : **26** (25 posées en avril 2026, + 1 palier ajouté depuis l'admin ; était 30 avant)

## Statistiques d'usage

Les niveaux réellement occupés se lisent sur la page admin `/analytics/levels` (RPC `get_level_stats`, migration 084). En août 2026, le meilleur joueur est au **niveau 10** et la projection de fin de saison plafonne autour du **niveau 15** : les paliers 16+ ne sont atteints par personne sur la saison 1.
