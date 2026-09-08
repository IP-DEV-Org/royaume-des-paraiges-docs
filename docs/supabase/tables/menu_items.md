# Table: menu_items

## Description

Un produit sur la carte d'un établissement. **Table centrale de la couche « Menus »**, introduite par la migration **095 (07/09/2026)** lors de la reprise de l'application `menu-ripaille` dans la base Royaume.

Elle remplace à elle seule les trois tables du modèle d'origine : `products` (catalogue global), `establishment_products` (pivot) et `establishment_variant_prices` (surcharge du prix de référence). Le repli est possible parce que l'arbitrage produit a changé : hors bières et softs, **un produit appartient à un seul établissement**.

Les données de l'ancien projet confirment que la séparation ne servait à rien : 365 produits pour 365 liaisons (aucun produit sur deux cartes) et **zéro** ligne de surcharge de prix.

> ⚠️ **Conséquence de sécurité.** L'ancien projet avait une faille active en production : la policy d'écriture de `products` était ouverte à tout compte `establishment`, qui pouvait donc modifier, retarifer et **supprimer** un produit affiché sur la carte d'un autre établissement. Elle disparaît ici **par construction**, en ne recréant pas la table globale qui la portait, et non par un correctif.

## Grain

Une ligne par produit et par établissement. `establishment_id` est `NOT NULL` : il n'existe pas d'item hors établissement.

## Les trois sources de descriptif

Un item tire son nom, sa description et son image d'exactement **une** des trois sources, garantie par `CHECK (num_nonnulls(beer_id, catalog_product_id, title) = 1)` :

| Source | Colonne | Pour quoi |
|---|---|---|
| Catalogue des bières | `beer_id` → `beers` | Les bières et cidres. Le descriptif vient de `beers`, la carte n'ajoute que prix, format et placement. |
| Catalogue partagé | `catalog_product_id` → `menu_catalog_products` | Les softs. Même principe. |
| Produit privé | `title` (+ `description`, `featured_image`…) | Tout le reste : restauration, cocktails, spiritueux, vins, goodies. |

Il n'y a **pas de table de surcharge**, parce que la ligne par établissement *est* l'item : le prix vit sur `menu_item_variants`, la disponibilité sur `is_active`, le placement sur `category_id` et `position`. Toute valeur spécifique à venir est une colonne de plus ici, pas une restructuration.

## Schema

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | bigint (identity) | Non | - | PK. |
| `establishment_id` | integer | Non | - | FK `establishments(id)` ON DELETE CASCADE. |
| `category_id` | bigint | Oui | - | FK `menu_categories(id)` ON DELETE **SET NULL**. `NULL` = disponible mais **hors carte affichée**. |
| `item_type_id` | bigint | Non | - | FK `menu_item_types(id)`. Classification transverse. |
| `beer_id` | integer | Oui | - | FK `beers(id)` ON DELETE **RESTRICT**. |
| `catalog_product_id` | bigint | Oui | - | FK `menu_catalog_products(id)` ON DELETE **RESTRICT**. |
| `title` | text | Oui | - | Renseigné uniquement pour un produit privé. |
| `description` | text | Oui | - | Surcharge locale du descriptif de la source. |
| `featured_image` | text | Oui | - | Chemin dans le bucket `menu-assets` (migration 094), pas une URL absolue. |
| `allergens` | text | Oui | - | |
| `precision` | text | Oui | - | Mention sous le titre : « 2 pers. », « 4 demi ». |
| `position` | integer | Non | 0 | Ordre dans la catégorie. |
| `is_active` | boolean | Non | true | `false` = à la carte mais en rupture. |
| `is_featured` | boolean | Non | false | Coup de cœur. |
| `added_at` | timestamptz | Non | now() | Date de mise à la carte. Reprise de `beers_establishments.added_at` au backfill. |
| `created_at` / `updated_at` | timestamptz | Non | now() | `updated_at` maintenu par `set_updated_at()`. |

## Le `category_id` nullable est load-bearing

Une bière peut être **disponible sans figurer sur la carte imprimée** : sa catégorie est nulle. C'est cette nuance qui rend inutile un second endroit où stocker la disponibilité, et c'est ce qui a permis de reprendre les 43 liaisons de `beers_establishments` sans leur inventer une catégorie. Voir [beers_establishments.md](./beers_establishments.md).

Il y a donc **deux niveaux de disponibilité**, distincts et tous deux utiles :

- absence de ligne : le produit n'est pas à la carte du tout ;
- `is_active = false` : il y est mais en rupture.

## Contraintes

- `CHECK (num_nonnulls(beer_id, catalog_product_id, title) = 1)` — exactement une source de descriptif.
- `CHECK (title IS NULL OR length(btrim(title)) > 0)` — un titre blanc satisferait `num_nonnulls` en produisant un item sans nom.
- `UNIQUE (establishment_id, beer_id) WHERE beer_id IS NOT NULL` — une bière ne peut apparaître qu'une fois par carte. **C'est cette contrainte qui force à fusionner les doublons de l'ancien projet** : « Delirium Red » et « Delirium Red 8° » sont deux lignes pour une seule bière, l'une en pression, l'autre en bouteille 75 cl. Elles deviennent un item à trois variantes.
- `UNIQUE (establishment_id, catalog_product_id) WHERE catalog_product_id IS NOT NULL` — idem pour les softs.
- `UNIQUE (establishment_id, category_id) WHERE is_featured AND category_id IS NOT NULL` — **un seul coup de cœur par catégorie** (migration 103). Les items hors carte sont exclus : les étoiler n'a aucun effet visible.

### Poser un coup de cœur : `set_menu_item_featured(p_item_id, p_featured)`

Ne pas écrire `is_featured` directement. Avec l'index seul, étoiler un produit dans une catégorie qui en a déjà un lèverait un `23505` et obligerait à désétoiler d'abord — ce n'est pas ce qu'on attend d'une étoile, on veut qu'elle se **déplace**. La fonction retire l'ancien et pose le nouveau **atomiquement**, ce que deux appels REST ne peuvent pas garantir.

`SECURITY INVOKER` : la RLS de `menu_items` s'applique normalement, `admin_has_feature('menus')` compris. Aucun privilège supplémentaire n'est accordé.

### Trigger `trg_menu_items_scope`

`BEFORE INSERT OR UPDATE`, fonction `enforce_menu_item_scope()`, errcode **`P0426`** :

- la catégorie doit appartenir au **même établissement** que l'item. Une clé étrangère composite l'exprimerait mieux, mais elle demanderait un index unique `(id, establishment_id)` sur `menu_categories` pour un gain nul ailleurs ;
- un item lié à une bière doit être de type `biere` ou `cidre`. Sans cette garde, un item lié pourrait être classé « cocktail » et sortirait du filtre de la vue `beers_establishments`, faisant disparaître la bière de l'app cliente sans que rien ne le signale.

## Cascades

- `establishment_id` **CASCADE** : supprimer un établissement emporte sa carte.
- `category_id` **SET NULL** : supprimer une catégorie ne doit pas emporter ses produits. Ils redeviennent disponibles mais hors carte, en attente d'un nouveau rangement.
- `beer_id` / `catalog_product_id` **RESTRICT** : supprimer une bière du catalogue ne doit pas retirer silencieusement un produit des cartes qui l'affichent. La suppression échoue, l'admin retire l'item d'abord.

## Index

| Index | Usage |
|---|---|
| `idx_menu_items_category (category_id, position)` | Rendu de la carte. |
| `idx_menu_items_establishment (establishment_id, position)` | Administration. |
| `idx_menu_items_beer_available (establishment_id, beer_id) WHERE beer_id IS NOT NULL AND is_active` | Porte la vue `beers_establishments`, seul accès chaud depuis les autres applications. |

## RLS

RLS active : **Oui**

| Policy | Action | Condition |
|---|---|---|
| `menu_items_admin_select` | SELECT | `profiles.role = 'admin'` |
| `menu_items_public_beer_availability` | SELECT | `beer_id IS NOT NULL AND is_active` (`authenticated`) : porte la vue [beers_establishments](./beers_establishments.md) depuis son passage en `security_invoker` (migration 105). Même prédicat que la vue, pas une ligne de plus. |
| `menu_items_feature_insert` | INSERT | `admin_has_feature('menus')` |
| `menu_items_feature_update` | UPDATE | `admin_has_feature('menus')` |
| `menu_items_feature_delete` | DELETE | `admin_has_feature('menus')` |

Même patron que la migration 070 sur les quêtes : « fonctionnalité active » est une barrière **dure en base**, pas seulement dans le middleware Next.js. Un admin dont la fonctionnalité `menus` est désactivée par un super-admin ne peut pas écrire, même par appel REST direct.

**Un gérant n'administre pas sa carte** : l'écriture est réservée au rôle `admin`. Aucune table d'appartenances n'a donc été créée, et `profiles.attached_establishment_id` n'est pas touché.

`anon` n'a **aucun grant** sur cette table : la carte publique passe par [get_public_menu](../functions/get_public_menu.md).

Tout compte connecté, client compris, lit en revanche les lignes « bière active » (toutes colonnes) via la policy ci-dessus : c'est du contenu de carte déjà public, plus la notion « disponible mais hors carte » que la vue exposait déjà.

## Exemples de requêtes

```sql
-- La carte d'un établissement, telle qu'affichée
SELECT mi.position,
       COALESCE(b.title, cp.title, mi.title) AS titre,
       mit.slug AS type
  FROM menu_items mi
  JOIN menu_item_types mit ON mit.id = mi.item_type_id
  LEFT JOIN beers b ON b.id = mi.beer_id
  LEFT JOIN menu_catalog_products cp ON cp.id = mi.catalog_product_id
 WHERE mi.establishment_id = 5
   AND mi.is_active
   AND mi.category_id IS NOT NULL
 ORDER BY mi.position;

-- Les produits disponibles mais absents de la carte affichée
SELECT * FROM menu_items
 WHERE establishment_id = 5 AND category_id IS NULL AND is_active;
```

## Voir aussi

- [menu_item_variants](./menu_item_variants.md) — les formats tarifés.
- [menu_catalog_products](./menu_catalog_products.md) — le catalogue partagé hors bières.
- [menu_item_types](./menu_item_types.md) — la classification transverse.
- [menu_categories](./menu_categories.md) — le regroupement éditorial.
- [beers_establishments](./beers_establishments.md) — la vue qui en découle.
