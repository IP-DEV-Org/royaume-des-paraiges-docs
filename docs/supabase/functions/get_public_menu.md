# Fonctions : get_public_menu / get_public_menu_item

Point d'entrée **unique** de la carte publique. Introduites par la migration **095 (07/09/2026)**.

```sql
get_public_menu(p_slug text) RETURNS jsonb
get_public_menu_item(p_slug text, p_item_id bigint) RETURNS jsonb
get_public_menu_establishments() RETURNS jsonb   -- migration 102
```

`STABLE`, `SECURITY DEFINER`, `search_path` fixé à `public, pg_temp`. `EXECUTE` accordé à `anon`, `authenticated` et `service_role`.

## Pourquoi une fonction plutôt qu'une lecture directe

L'application d'origine lisait ses tables directement, avec une policy `select using (true)` sur chacune. Ce modèle a deux défauts que la fonction supprime :

**Sécurité.** Les tables `menu_*` n'ont **aucun grant `anon`** : il n'existe pas de chemin de lecture publique en dehors de ces deux fonctions. Le mode de fuite le plus courant, la policy oubliée sur une table fille (les variantes, les options), devient impossible.

**Performance.** La carte complète part en **un aller-retour** au lieu de quatre ou cinq. Le rendu d'une carte demande l'établissement, ses catégories, ses items, leurs variantes, leurs groupes d'options, ses formules et ses événements.

C'est le patron déjà en place sur `get_analytics_*` et `get_email_report_payload`, en plus étroit : aucune donnée d'administration ne transite.

## Garde-fous

- Aucun SQL dynamique. Les paramètres ne servent qu'en `WHERE`.
- La fonction ne peut retourner que du contenu **actif** et rattaché au slug demandé.
- **`get_public_menu_item` vérifie le slug en plus de l'id.** Sans lui, l'id seul permettrait d'énumérer les items de toutes les cartes.
- Slug inconnu, item inactif, item hors carte : `NULL`. Jamais d'erreur, jamais de fuite d'existence.

## Ce qui est exclu du rendu

Un item dont `category_id IS NULL` est **disponible mais hors carte affichée** : il n'apparaît dans aucune des deux fonctions, alors qu'il reste visible dans la vue [beers_establishments](../tables/beers_establishments.md). Voir [menu_items](../tables/menu_items.md).

## Forme du retour

```jsonc
{
  "establishment": {
    "title": "...", "slug": "...", "short_description": "...", "description": "...",
    "logo": "...", "featured_image": "...",
    "address": { "line_1": "...", "line_2": "...", "zipcode": "...", "city": "...", "country": "..." },
    "happy_hour": { "start": "17:00:00", "end": "20:00:00" }   // null si non configuré
  },
  "sections":   [ { "id", "title", "description" } ],                 // migration 107
  "categories": [ { "id", "parent_id", "section_id", "title", "description", "position" } ],
  "items": [ {
    "id", "category_id", "type",          // type = menu_item_types.slug
    "title", "description", "featured_image", "allergens", "precision",
    "is_featured", "position",
    "beer": { "id", "abv", "ibu", "brewery" },   // null si l'item n'est pas une bière
    "variants":      [ { "label", "price", "is_happy_hour" } ],
    "option_groups": [ { "title", "min_select", "max_select",
                         "options": [ { "label", "extra_price" } ] } ]
  } ],
  "formulas": [ { "title", "description", "tiers": [ { "label", "price" } ] } ],
  "events":   [ { "title", "content", "featured_image", "external_url" } ]
}
```

## Sections (migration 107)

Une [section](../tables/menu_sections.md) est un chapitre qui regroupe des catégories racines par leur `section_id`. Elle n'a pas de position : la carte la place **à l'emplacement de sa première catégorie visible**, dans l'ordre des `position` de catégories. Une section dont aucune catégorie racine n'est active n'est pas renvoyée.

## Résolution du descriptif

Le titre suit la source, sans surcharge possible : `COALESCE(beers.title, menu_catalog_products.title, menu_items.title)`. Le catalogue fait foi.

Description, image, allergènes et précision acceptent en revanche une **surcharge locale** : `COALESCE(menu_items.<col>, <source>.<col>)`. Un établissement peut préciser sa propre description sans toucher au catalogue partagé.

## `get_public_menu_establishments` (migration 102)

Sert l'accueil de l'application de cartes : les établissements **ayant au moins un
produit actif ET placé dans une catégorie**. Une bière seulement disponible
(`category_id` nul) ne fait pas une carte, et lister un établissement sans carte
mènerait à une page vide.

Le filtre ne pouvait pas vivre côté application : les tables `menu_*` ne lui sont
pas accessibles. Un slug absent de cette liste reste **atteignable en direct** —
l'établissement existe, sa carte est simplement vide.

## Exemples

```sql
SELECT get_public_menu('delirium-cafe-strasbourg');
SELECT get_public_menu_item('la-ripaille', 601);

-- Slug inconnu : NULL, pas une erreur
SELECT get_public_menu('nexiste-pas') IS NULL;   -- true
```
