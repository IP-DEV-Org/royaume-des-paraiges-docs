# Table: establishments

## Description

Etablissements partenaires.

## Schema

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | integer | Non | - | ID |
| `title` | varchar | Non | - | Nom de l'etablissement |
| `line_address_1` | varchar | Oui | - | Adresse ligne 1 |
| `line_address_2` | varchar | Oui | - | Adresse ligne 2 |
| `zipcode` | varchar | Oui | - | Code postal |
| `city` | varchar | Oui | - | Ville |
| `country` | varchar | Oui | - | Pays |
| `short_description` | text | Oui | - | Description courte |
| `description` | text | Oui | - | Description complete |
| `featured_image` | text | Oui | - | URL de l'image principale |
| `logo` | text | Oui | - | URL du logo |
| `anniversary` | date | Oui | - | Date anniversaire |
| `slug` | text | Non | - | Identifiant d'URL de la carte publique (`/[slug]`) et cible des liens courts `redirect.auxparaiges.fr`. UNIQUE, `CHECK (slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$')`. Migration **094**. |
| `happy_hour_start` | time | Oui | - | Début du happy hour, heure locale. `NULL` = pas de happy hour. Migration **095**. |
| `happy_hour_end` | time | Oui | - | Fin du happy hour. |
| `created_at` | timestamptz | Oui | now() | Date de creation |
| `updated_at` | timestamptz | Oui | now() | Date de derniere modification (auto via trigger) |

## Cle primaire

- `id`

## Relations

### Tables liees

| Table | Colonne |
|-------|---------|
| `beers_establishments` | `establishment_id` | (vue, migration 096) |
| `menu_categories` | `establishment_id` | |
| `menu_items` | `establishment_id` | |
| `menu_option_groups` | `establishment_id` | |
| `menu_formulas` | `establishment_id` | |
| `menu_establishment_events` | `establishment_id` | |
| `news_establishments` | `establishment_id` |
| `receipts` | `establishment_id` |
| `gains` | `establishment_id` |
| `spendings` | `establishment_id` |

## RLS

RLS active: **Oui**

## Exemples de requetes

```sql
-- Tous les etablissements avec leurs bieres
SELECT e.*,
       array_agg(b.title) as beers
FROM establishments e
LEFT JOIN beers_establishments be ON e.id = be.establishment_id
LEFT JOIN beers b ON be.beer_id = b.id
GROUP BY e.id;

-- Etablissements par ville
SELECT * FROM establishments
WHERE city = 'Metz';
```

## Statistiques

- Lignes: **7**

## Foreign Keys entrantes

| Table | Colonne | ON DELETE |
|-------|---------|-----------|
| `receipts` | `establishment_id` | RESTRICT |
| `gains` | `establishment_id` | RESTRICT |
| `spendings` | `establishment_id` | RESTRICT |
| `coupon_templates` | `establishment_id` | RESTRICT |
| `comments` | `establishment_id` | CASCADE |
| `beers_establishments` | `establishment_id` | CASCADE |
| `news_establishments` | `establishment_id` | CASCADE |

## Triggers

| Trigger | Event | Description |
|---------|-------|-------------|
| `set_establishments_updated_at` | BEFORE UPDATE | Met a jour `updated_at` automatiquement |

## Ameliorations futures

### Geolocalisation (PostGIS)

Ajout prevu de coordonnees GPS pour permettre :
- Recherche de proximite ("etablissements a moins de X km")
- Affichage sur carte
- Tri par distance

```sql
-- Structure prevue
ALTER TABLE establishments
  ADD COLUMN latitude DECIMAL(10, 8),
  ADD COLUMN longitude DECIMAL(11, 8),
  ADD COLUMN location GEOGRAPHY(POINT, 4326);

CREATE INDEX idx_establishments_location
  ON establishments USING GIST(location);
```

```sql
-- Exemple de requete de proximite
SELECT *, ST_Distance(location, ST_MakePoint(6.175, 49.119)::geography) as distance
FROM establishments
WHERE ST_DWithin(location, ST_MakePoint(6.175, 49.119)::geography, 1000)
ORDER BY distance;
```

## Slug et happy hour (migrations 094 et 095)

`slug` est **dérivé du titre** à la création, via `menu_slugify(text)`. Les 7 établissements ont été backfillés : `aux-paraiges`, `le-troubadour`, `le-garage-des-paraiges`, `la-grange-des-paraiges`, `delirium-cafe-strasbourg`, `la-chapelle`, `la-ripaille`.

> ⚠️ Le slug est **modifiable, mais le changer casse les QR codes déjà imprimés** qui pointent dessus, directement ou via un lien court de `/links`.

`happy_hour_start` / `happy_hour_end` portent la plage d'un happy hour quotidien. Elle sert à afficher la section au bon moment sur la carte publique ; les tarifs eux-mêmes sont portés par `menu_item_variants.is_happy_hour`. Deux colonnes plutôt qu'une table : le happy hour est une plage unique et quotidienne. Le jour où il varie selon le jour de la semaine, ce sera une table, et ces colonnes s'y videront.
