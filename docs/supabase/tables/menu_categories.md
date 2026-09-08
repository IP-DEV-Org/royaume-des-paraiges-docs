# Table: menu_categories

## Description

Le **regroupement éditorial** des items sur la carte d'un établissement. Introduite par la migration **095 (07/09/2026)**.

Propre à l'établissement : « Happy hour » chez l'un n'a rien à voir avec celui d'un autre. Une catégorie mélange volontiers les familles de produits — chez le Delirium, « Happy hour » contenait des softs, des cocktails et un vin. La classification transverse est portée par [menu_item_types](./menu_item_types.md), pas par la catégorie.

## Hiérarchie à deux niveaux

« Bières Bouteilles » contient « Bière blonde », « Bière fruitée », « Bière sans alcool ». La profondeur est **bornée par trigger** : rien n'empêcherait sinon une catégorie petite-fille que l'affichage en accordéon ne sait pas rendre.

## Schema

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | bigint (identity) | Non | - | PK. |
| `establishment_id` | integer | Non | - | FK `establishments(id)` ON DELETE CASCADE. |
| `parent_id` | bigint | Oui | - | FK `menu_categories(id)` ON DELETE CASCADE. `NULL` = catégorie racine. |
| `section_id` | bigint | Oui | - | FK [menu_sections](./menu_sections.md)`(id)` ON DELETE **SET NULL**. Chapitre de la carte ; réservé aux catégories racines (migration 107). |
| `title` | text | Non | - | |
| `description` | text | Oui | - | Bloc de texte affiché sous le titre de section. |
| `position` | integer | Non | 0 | |
| `is_active` | boolean | Non | true | |
| `created_at` / `updated_at` | timestamptz | Non | now() | |

## Un chapitre au-dessus de la racine

Depuis la migration **107**, une catégorie racine peut rejoindre une [section](./menu_sections.md) : un chapitre de la carte qui rassemble plusieurs accordéons (« Les Cocktails » du Garage regroupe six familles). Une sous-catégorie ne porte jamais de section, elle suit son parent (trigger `trg_menu_categories_section`, errcode `P0426`).

## Une catégorie peut être un bloc de texte

`description` peut porter une catégorie **sans aucun item** : c'est ainsi que La Ripaille rend sa section « Notre Histoire », extraite de la carte et affichée en bas de page. Ne pas supposer qu'une catégorie vide est une erreur de saisie.

## Contraintes

- `CHECK (parent_id IS DISTINCT FROM id)`.
- Trigger `trg_menu_categories_depth`, fonction `enforce_menu_category_depth()`, errcode **`P0426`** :
  - une catégorie dont le parent a déjà un parent est refusée (deux niveaux maximum) ;
  - une sous-catégorie rattachée à un **autre établissement** est refusée : elle ferait apparaître des items étrangers sur la carte, c'est un défaut de cloisonnement et pas une erreur de saisie.
- Trigger `trg_menu_categories_section`, fonction `enforce_menu_category_section()`, errcode **`P0426`** (migration 107) : une section ne se pose que sur une catégorie racine, et du même établissement.

## Cascade

Attention à l'asymétrie voulue : supprimer une catégorie **CASCADE** sur ses sous-catégories, mais **SET NULL** sur ses items ([menu_items](./menu_items.md)). Les produits ne disparaissent pas, ils redeviennent disponibles hors carte.

## Index

- `idx_menu_categories_establishment (establishment_id, position)`
- `idx_menu_categories_parent (parent_id) WHERE parent_id IS NOT NULL`
- `idx_menu_categories_section (section_id) WHERE section_id IS NOT NULL`

## RLS

RLS active : **Oui**. Lecture admin, écriture soumise à `admin_has_feature('menus')`.
