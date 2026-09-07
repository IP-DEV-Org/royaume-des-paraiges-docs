# Table: menu_item_variants

## Description

Les **formats tarifés** d'un item de carte. Introduite par la migration **095 (07/09/2026)**.

Un item sans variante n'a pas de prix affichable : la carte publique l'ignore.

## Variante ou option ?

La distinction est nette, et vient de l'usage réel constaté dans l'ancien projet :

| | Variante | Option |
|---|---|---|
| Quoi | Un **format tarifé** | Un **choix**, avec supplément facultatif |
| Sur la carte | Produit sa propre ligne de prix | Listée sous l'item, sans ligne propre |
| Exemple | `25 cl 4,10 €` / `50 cl 7,50 €` | Sauces, accompagnements, parfums d'un jus |

Concrètement, les parfums d'un jus de fruits vendus au même prix sont un groupe d'options, pas trois variantes : la carte affiche une ligne « Jus de fruits 4,00 € » avec les parfums dessous, plutôt que trois lignes identiques.

## Schema

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | bigint (identity) | Non | - | PK. |
| `menu_item_id` | bigint | Non | - | FK `menu_items(id)` ON DELETE CASCADE. |
| `label` | text | Oui | - | `NULL` = produit simple, sans déclinaison. Sinon le **format seul** : « 25 cl », « 50 cl ». |
| `price` | numeric(10,2) | Oui | - | `NULL` = prix non communiqué, affiché « — ». Distinct de `0`, qui est la gratuité. |
| `is_happy_hour` | boolean | Non | false | Tarif happy hour. |
| `position` | integer | Non | 0 | Ordre d'affichage. |
| `created_at` / `updated_at` | timestamptz | Non | now() | `updated_at` via `set_updated_at()`. |

## Le happy hour

> ⚠️ **Ne jamais écrire « happy hour » dans `label`.** C'est le rôle de `is_happy_hour`. L'ancien projet le faisait en texte libre, d'où 17 variantes de bières portant `happy hour 50cl`, `Happy Hour 50 cL`, `happy hour 50 cl`.

`is_happy_hour` permet de **reconstituer la section « Happy hour » de la carte par filtre**, sans dupliquer les produits. La plage horaire vit sur `establishments.happy_hour_start` / `happy_hour_end`, et sert à afficher la section au bon moment, comme la carte remonte déjà la formule du midi entre 11 h et 15 h.

Ce que ce modèle résout, et qu'une simple mise en variante ne résolvait pas : dans l'ancien projet, la catégorie « Happy hour » du Delirium contenait **11 lignes dont 2 produits proposés uniquement au happy hour** (`Punch Planteur`, `Vin d'Alsace`). Dissoudre la catégorie sans drapeau les aurait laissés sans catégorie d'accueil.

Contraintes : `CHECK (price IS NULL OR price >= 0)`.

## Index

- `idx_menu_item_variants_item (menu_item_id, position)`.

## RLS

RLS active : **Oui**. Lecture admin, écriture soumise à `admin_has_feature('menus')`. Aucun grant `anon` : la carte publique passe par [get_public_menu](../functions/get_public_menu.md).

## Exemples de requêtes

```sql
-- Tous les tarifs happy hour d'un établissement
SELECT COALESCE(b.title, cp.title, mi.title) AS produit, v.label, v.price
  FROM menu_item_variants v
  JOIN menu_items mi ON mi.id = v.menu_item_id
  LEFT JOIN beers b ON b.id = mi.beer_id
  LEFT JOIN menu_catalog_products cp ON cp.id = mi.catalog_product_id
 WHERE mi.establishment_id = 5 AND v.is_happy_hour
 ORDER BY produit;
```
