# Table: menu_item_types

## Description

Les **familles de produits** d'une carte. Introduite par la migration **095 (07/09/2026)**.

Table de référence et **non un enum** : les familles évoluent (un établissement ouvre une cave à vins, un autre vend du merch), et ajouter une valeur ne doit pas demander une migration.

## À ne pas confondre avec deux voisines

**Ce n'est pas une catégorie.** La catégorie ([menu_categories](./menu_categories.md)) est le regroupement **éditorial** d'un établissement : « Happy hour », « Nouveautés », « Bières Bouteilles ». Elle mélange volontiers les familles. Le type est une classification **transverse** : il dit ce qu'*est* un produit, quel que soit l'endroit où l'établissement l'a rangé.

**Ce n'est pas `consumption_type`.** L'enum `consumption_type` sert au **scan**, aux tickets et aux quêtes. Les deux mondes ne se recouvrent pas : « Goodies » n'a pas de consommation associée, et le scan ne distingue pas un shooter d'un digestif. La colonne `consumption_type` ci-dessous fait le pont pour les familles où la correspondance existe, **sans jamais contraindre la saisie de la carte**.

## Schema

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | bigint (identity) | Non | - | PK. |
| `slug` | text | Non | - | UNIQUE, `CHECK (slug ~ '^[a-z0-9]+(_[a-z0-9]+)*$')`. Identifiant stable, exposé par les RPC publiques. |
| `label` | text | Non | - | Libellé affiché en administration. |
| `consumption_type` | consumption_type | Oui | - | Pont facultatif vers l'enum du scan. `NULL` quand aucune correspondance ne s'applique. |
| `position` | integer | Non | 0 | Ordre d'affichage en administration. |
| `is_active` | boolean | Non | true | |
| `created_at` / `updated_at` | timestamptz | Non | now() | |

## Seed

14 lignes, dérivées des 52 catégories réellement présentes dans les cartes du Delirium et de La Ripaille.

| slug | label | consumption_type |
|---|---|---|
| `biere` | Bière | `biere` |
| `cidre` | Cidre | `biere` |
| `cocktail` | Cocktail | `cocktail` |
| `spiritueux` | Spiritueux | `alcool` |
| `shooter` | Shooter | `alcool` |
| `aperitif` | Apéritif | `alcool` |
| `digestif` | Liqueur et digestif | `alcool` |
| `vin` | Vin | `alcool` |
| `soft` | Soft | `soft` |
| `boisson_chaude` | Boisson chaude | `boisson_chaude` |
| `restauration` | Restauration | `restauration` |
| `dessert` | Dessert | `restauration` |
| `boucherie` | Viande à la découpe | `boucherie` |
| `goodies` | Goodies | `NULL` |

Le **cidre est rattaché à `biere`** et non à `alcool` : le catalogue Royaume range déjà les cidres dans `beers` (Magners, Wild Wave Cider), la carte doit dire la même chose que lui.

`goodies` est la seule famille sans `consumption_type` : le merchandising n'est pas une consommation.

## Contrainte croisée

Le trigger `trg_menu_items_scope` sur [menu_items](./menu_items.md) exige qu'un item lié à une bière soit de type `biere` ou `cidre`. Renommer ou désactiver ces deux slugs casserait donc l'insertion de toute bière de carte.

## RLS

RLS active : **Oui**. Lecture admin ; écriture **réservée au super admin** (`is_super_admin()`, policies `menu_item_types_super_admin_*`) depuis la migration **109** : référentiel partagé par toutes les cartes, une famille ne se modifie pas depuis un établissement.
