# Tables satellites de la couche Menus : options, formules, événements

Sept tables secondaires de la couche « Menus », introduites par la migration **095 (07/09/2026)**. Elles sont reprises quasi telles quelles du modèle `menu-ripaille`, où elles étaient déjà correctement cloisonnées par établissement : il n'y avait rien à corriger.

Regroupées ici plutôt qu'en sept fichiers parce qu'elles n'ont ni subtilité de modélisation ni piège d'exploitation. Les cinq tables centrales ont chacune leur page : [menu_items](./menu_items.md), [menu_item_variants](./menu_item_variants.md), [menu_catalog_products](./menu_catalog_products.md), [menu_item_types](./menu_item_types.md), [menu_categories](./menu_categories.md). Les chapitres qui regroupent des catégories ont aussi la leur : [menu_sections](./menu_sections.md).

Toutes ont la même RLS : lecture `role = 'admin'`, aucun grant `anon`. En écriture, depuis la migration **109**, elles suivent le périmètre de [`admin_can_edit_menu`](../functions/admin_can_edit_menu.md) (un admin ne modifie que la carte de son établissement de rattachement) : `menu_option_groups`, `menu_formulas` et `menu_establishment_events` testent leur `establishment_id` (policies `*_scoped_*`) ; `menu_options` remonte à son groupe, `menu_formula_tiers` à sa formule, et `menu_item_option_groups` exige que l'item **et** le groupe soient du même admin. `menu_events`, partagée entre cartes, passe en écriture super admin (`is_super_admin()`, policies `menu_events_super_admin_*`). Toutes celles qui portent `updated_at` ont leur trigger `set_updated_at()`.

## Options

Sauces, accompagnements, parfums. L'option est un **choix**, éventuellement avec supplément ; elle n'a pas de ligne de prix propre sur la carte, contrairement à la variante. Voir [menu_item_variants](./menu_item_variants.md) pour la distinction.

### `menu_option_groups`

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | bigint (identity) | Non | - | PK. |
| `establishment_id` | integer | Non | - | FK `establishments(id)` ON DELETE CASCADE. |
| `title` | text | Non | - | « Sauces », « Accompagnements », « Parfums ». |
| `min_select` | integer | Non | 0 | `CHECK (>= 0)`. |
| `max_select` | integer | Oui | - | `NULL` = **pas de plafond** (les moutardes spéciales de La Ripaille se cumulent). |
| `position` | integer | Non | 0 | |

Contrainte : `CHECK (max_select IS NULL OR max_select >= min_select)`.

### `menu_options`

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | bigint (identity) | Non | - | PK. |
| `option_group_id` | bigint | Non | - | FK `menu_option_groups(id)` ON DELETE CASCADE. |
| `label` | text | Non | - | |
| `extra_price` | numeric(10,2) | Oui | - | Supplément. `NULL` ou `0` = compris dans le prix de l'item. |
| `position` | integer | Non | 0 | |

### `menu_item_option_groups`

Pivot `menu_items` × `menu_option_groups`. PK composite `(menu_item_id, option_group_id)`, cascade des deux côtés, plus une colonne `position`.

## Formules

Menus à paliers. Un seul jeu existe à la reprise : la formule du midi de La Ripaille, à deux paliers.

### `menu_formulas`

`id`, `establishment_id` (CASCADE), `title`, `description`, `position`, `is_active`, horodatages.

### `menu_formula_tiers`

`id`, `formula_id` (CASCADE), `label` (« Entrée + Plat », « Entrée + Plat + Dessert »), `price` numeric(10,2) `CHECK (>= 0)`, `position`, horodatages.

> La carte publique remonte le bloc formules en haut de page **entre 11 h et 15 h heure de Paris**, et le redescend le reste du temps. Cette logique vit côté application, pas en base.

## Événements

Encart à image de fond redirigeant vers un lien externe.

### `menu_events`

`id`, `title`, `content`, `featured_image`, `external_url`, `is_active`, `position`, horodatages.

### `menu_establishment_events`

Pivot `establishments` × `menu_events`. PK composite, cascade des deux côtés, plus `position`.

> ⚠️ **Table dédiée, et non une fusion dans `news`.** La passation recommandait de fusionner, en ajoutant `external_url`, `is_active` et `position` à `news`. Décision contraire : l'actualité du programme de fidélité s'adresse aux Compagnons dans l'app, l'événement de carte au client assis à table. Deux audiences, deux tables. Fusionner ferait apparaître chaque actualité sur toutes les cartes, et réciproquement.
