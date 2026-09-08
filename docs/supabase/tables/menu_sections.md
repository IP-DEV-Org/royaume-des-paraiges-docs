# Table: menu_sections

## Description

Le **chapitre** de la carte d'un établissement : un titre, au besoin une phrase, et des catégories racines qui s'y rattachent. Introduite par la migration **107 (08/09/2026)**.

Née de la carte du Garage des Paraiges : onze catégories racines, dont six familles de cocktails (Incontournables, Classiques, Créations, Mocktails, Granités, Spritz) à rassembler. La [hiérarchie à deux niveaux](./menu_categories.md#hiérarchie-à-deux-niveaux) ne le permettait pas : une sous-catégorie s'affiche en intertitre dans l'accordéon de son parent, et une catégorie qui a déjà des sous-catégories (les Shooters, les Spiritueux) ne pourrait pas rejoindre un groupe. La section est une notion **au-dessus** de la catégorie, pas en dessous.

## Ce qu'une section n'a pas, et pourquoi

- **Pas d'item.** Un chapitre n'est pas un rayon : les produits restent dans les catégories, et tout ce qui raisonne en catégories (recherche, coup de cœur, fiche produit, index `menu_items_one_featured_per_category`) est inchangé.
- **Pas de position.** La carte se classe sur une seule échelle, celle de `menu_categories.position`. La section s'affiche **à l'emplacement de sa première catégorie visible** et rassemble les autres derrière elle. Deux échelles auraient permis des états incohérents (une section classée après ses propres catégories) et un champ de plus à saisir.
- **Pas de `is_active`.** Masquer se fait catégorie par catégorie ; une section dont aucune catégorie racine n'est visible n'est pas renvoyée par [get_public_menu](../functions/get_public_menu.md).

## Schema

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | bigint (identity) | Non | - | PK. |
| `establishment_id` | integer | Non | - | FK `establishments(id)` ON DELETE CASCADE. |
| `title` | text | Non | - | `CHECK (btrim(title) <> '')`. |
| `description` | text | Oui | - | Phrase sous le titre du chapitre, sur la carte publique. |
| `created_at` / `updated_at` | timestamptz | Non | now() | Trigger `trg_menu_sections_updated_at`. |

## Rattachement

`menu_categories.section_id` (FK **ON DELETE SET NULL**) : supprimer un chapitre remet ses catégories à plat, il n'emporte rien. Même asymétrie voulue que les items d'une catégorie supprimée.

Trigger `trg_menu_categories_section` (fonction `enforce_menu_category_section()`, errcode **`P0426`**) :

- `MENU_CATEGORY_SECTION_ON_CHILD` : seule une catégorie **racine** peut rejoindre une section, une sous-catégorie suit son parent ;
- `MENU_SECTION_CROSS_ESTABLISHMENT` : la section doit appartenir au même établissement que la catégorie.

La fonction est `SECURITY DEFINER`, exécution révoquée à `anon` et `authenticated` (ce n'est pas une RPC, cf. migration 097).

## Rendu sur la carte publique

La section est elle-même un accordéon, plus marqué qu'une catégorie, qui liste ses catégories en lignes repliables. Choisi le 08/09/2026 parmi quatre propositions (intertitre, section pliable, onglets, ou la hiérarchie existante).

## Index

- `idx_menu_sections_establishment (establishment_id)`
- `idx_menu_categories_section (section_id) WHERE section_id IS NOT NULL`

## RLS

RLS active : **Oui**. Même patron que les autres tables `menu_*` : lecture admin, écriture soumise à `admin_has_feature('menus')`, aucun grant `anon`.

## Données

Migration **108** : le chapitre « Les Cocktails » du Garage des Paraiges, avec ses six familles.
