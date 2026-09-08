# Table: menu_catalog_products

## Description

Le **catalogue partagé des produits hors bières**. Introduite par la migration **095 (07/09/2026)**.

Les bières ont déjà leur catalogue, `beers`, administré depuis `/content/beers`. Cette table est le catalogue partagé des **softs** à date, et de toute famille qu'on déciderait de partager plus tard : ajouter « les vins sont partagés » ne demandera que des lignes, pas de DDL.

## Ce qu'elle porte, et ce qu'elle ne porte pas

Elle ne porte **que le descriptif** : nom, description, image, allergènes. **Jamais le prix, jamais le format** : ils sont propres à chaque établissement et vivent sur [menu_item_variants](./menu_item_variants.md).

C'est ce découpage qui rend inutile une table de surcharge. L'ancien projet en avait une (`establishment_variant_prices`, surcharge du `default_price`) : elle n'a **jamais reçu une seule ligne**, parce que ce n'est pas ainsi qu'une carte fonctionne. Chaque établissement définit ses propres formats avec ses propres prix : le Delirium vend une pression en 25 et 50 cl, un autre la même référence en bouteille 33 cl.

## Règle d'entrée

**Tout soft entre au catalogue, même servi par un seul établissement.** Sinon le deuxième établissement qui l'ajoute demain le ressaisit à sa façon et recrée exactement la divergence que le catalogue existe pour empêcher.

Le catalogue compte **21 softs** à la reprise, dont 5 seulement servis par les deux établissements, puis **25** après la carte du Garage des Paraiges (migration **104**, 08/09/2026).

## Schema

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | bigint (identity) | Non | - | PK. |
| `item_type_id` | bigint | Non | - | FK `menu_item_types(id)`. |
| `title` | text | Non | - | Nom canonique, commun à tous les établissements. |
| `description` | text | Oui | - | |
| `featured_image` | text | Oui | - | Chemin dans le bucket `menu-assets`, pas une URL absolue. |
| `allergens` | text | Oui | - | |
| `precision` | text | Oui | - | Mention sous le titre : « 2 pers. », « 4 demi ». |
| `is_active` | boolean | Non | true | `false` masque le produit sur **toutes** les cartes. |
| `created_at` / `updated_at` | timestamptz | Non | now() | |

## Arbitrages de la reprise

Quatre cas ont demandé une décision produit plutôt qu'un rapprochement automatique :

| Cas | Arbitrage |
|---|---|
| « Coca-Cola / Coca Zéro », une ligne chez La Ripaille | Scindée en **deux produits**, tous deux mis à sa carte |
| « Jus de Fruits (Orange, Pomme, Ananas, Tomate) » | Un produit à **parfums en options**, la tomate devient un produit distinct |
| `Ice Tea` chez La Ripaille, `Fuze Tea` au Delirium | **Deux produits** : marques différentes |
| « Carola Rouge » 33 cl et « Carola rouge » 1 L | **Fusionnés** : un produit, deux variantes de format |

La carte du Garage (migration 104) applique la même règle de marque : « Gingerbeer Fever-Tree », « Iced-tea Liness », « Limonade Liness » et « Vittel » deviennent **quatre nouveaux produits** plutôt que des réemplois de `Ginger Beer`, `Ice Tea`, `Limonade` et `Eau minérale`. La marque fait partie du nom affiché, et le titre ne se surcharge pas par établissement. Un « Supplément sirop » à 0,10 € reste en revanche un item privé : ce n'est pas un soft, c'est un supplément.

## RLS

RLS active : **Oui**. Lecture admin ; écriture **réservée au super admin** (`is_super_admin()`, policies `menu_catalog_products_super_admin_*`) depuis la migration **109** : un produit du catalogue s'affiche sur plusieurs cartes, le retitrer depuis un établissement modifierait celles des autres. Aucun écran admin n'y écrit à date (les produits entrent par migration).
