# Table: available_periods

Calendrier des périodes (start_date / end_date inclusives) par `period_type`.
Seul consommateur : `expire_quest_progress()` (join sur `period_type` +
`period_identifier`, expire les `quest_progress` `in_progress` dont
`end_date < CURRENT_DATE`).

> ⚠️ **Convention semaine = ISO 8601 (lundi→dimanche), obligatoire.**
> `period_identifier` doit correspondre **exactement** à ce que produit
> `get_period_identifier` (`to_char(date,'IYYY-"W"IW')`) et `get_period_bounds`,
> qui sont tous deux ISO. Les lignes `weekly` étaient initialement seedées en
> semaines **dimanche→samedi**, ce qui décalait l'expiration d'un jour **chaque
> dimanche** (la semaine en cours était expirée trop tôt et le front
> l'affichait « expirée » via `get_user_quests`). Corrigé par la migration
> `042_available_periods_weekly_iso_alignment` : `weekly` régénérée avec
> `start_date` = lundi, `end_date` = dimanche. **Toute reseed future doit rester
> ISO** (sinon le bug du dimanche revient). `monthly` (1er → dernier jour du
> mois) et `yearly` (01-01 → 31-12) sont indépendants de la convention semaine.

## Informations

| Propriete | Valeur |
|-----------|--------|
| **Schema** | `public` |
| **RLS** | Active |
| **Lignes** | 130 |

## Colonnes

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | `bigint` | Non | - | - |
| `period_type` | `character varying(20)` | Non | - | - |
| `period_identifier` | `character varying(20)` | Non | - | - |
| `start_date` | `date` | Non | - | - |
| `end_date` | `date` | Non | - | - |
| `created_at` | `timestamp with time zone` | Oui | now() | - |

## Cles primaires

- `id`

## Relations (Foreign Keys)

Aucune relation definie.
