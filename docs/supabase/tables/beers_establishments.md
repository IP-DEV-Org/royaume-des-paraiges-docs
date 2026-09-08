# Vue: beers_establishments

> ⚠️ **Ce n'est plus une table.** Depuis la migration **096 (07/09/2026)**, `beers_establishments` est une **vue** sur [menu_items](./menu_items.md). L'ancienne table, un temps conservée sous le nom `beers_establishments_legacy`, a été supprimée par la migration **101** une fois l'import des cartes validé.
>
> Attention au nommage : un `DROP ... CASCADE` sur `beers_establishments` emporterait la vue dont dépendent le front et le dashboard.

## Description

Bières disponibles par établissement, **déduites des cartes**.

« Cette bière est disponible à cet établissement » est entièrement déductible de « cette bière est à la carte de cet établissement ». Le fait est le même, il ne doit exister qu'à un seul endroit. Un fait déduit est une vue, pas une table maintenue par trigger : un trigger a des trous (changement de `beer_id`, bascule de `is_active`, suppression en cascade, écriture directe résiduelle) et les deux copies finissent par diverger.

## Définition

```sql
CREATE VIEW beers_establishments AS
SELECT mi.id, mi.beer_id, mi.establishment_id, mi.added_at, mi.created_at
FROM menu_items mi
WHERE mi.beer_id IS NOT NULL AND mi.is_active;
```

Portée par l'index partiel `idx_menu_items_beer_available (establishment_id, beer_id) WHERE beer_id IS NOT NULL AND is_active`.

## Schema

| Colonne | Type | Description |
|---------|------|-------------|
| `id` | bigint | Identifiant de l'**item de carte**, et non plus d'une ligne de liaison. |
| `beer_id` | integer | Vient de `menu_items.beer_id`. |
| `establishment_id` | integer | Vient de `menu_items.establishment_id`. |
| `added_at` | timestamptz | Date de mise à la carte. Les 43 valeurs de l'ancienne table ont été reprises au backfill. |
| `created_at` | timestamptz | |

## Lecture seule

**Pour rendre une bière disponible, créer un `menu_items`**, pas une ligne ici.

Une bière peut être disponible **sans figurer sur la carte affichée** : son item a `category_id IS NULL`. Elle apparaît alors dans cette vue mais pas dans [get_public_menu](../functions/get_public_menu.md). C'est exactement ce qui a permis de reprendre les 43 liaisons existantes sans leur inventer une catégorie.

Corollaire : **retirer une bière de la carte la rend indisponible**, et inversement. Les deux gestes n'en font plus qu'un. C'est voulu.

## Qui lit, qui écrivait

Avant la bascule, seule l'application **serveurs** écrivait (`addBeerToEstablishment`, `removeBeerFromEstablishment`). Cet écran est retiré, le champ est donc libre. Le front et le dashboard ne font que lire, en `SELECT` : leur code n'a pas changé d'une ligne.

| Application | Usage |
|---|---|
| Front (Expo) | `beerService.getByEstablishment`, `countByEstablishment`, filtre `availableAt` |
| Admin | `contentService.getBeersEstablishments`, `getBeersByEstablishment`, `getEstablishmentsByBeer` |
| Waiters | `getEstablishmentBeers` (écran retiré) |

> ⚠️ **Point de vigilance PostgREST.** `getEstablishmentBeers()` côté serveurs utilisait l'embed `beers(*, brewery:breweries(*))`. PostgREST résout les relations d'une vue via ses colonnes d'origine, et `menu_items.beer_id` porte bien une clé étrangère vers `beers` : l'embed devrait continuer de fonctionner. À vérifier en conditions réelles avant de supprimer le legacy, ou à considérer comme sans objet si l'écran a déjà été retiré.

## RLS

La vue est en **`security_invoker`** depuis la migration **105** (08/09/2026), comme toutes les vues du projet depuis les migrations `security_*` de mai 2026 : elle lit `menu_items` avec les droits de l'appelant, et c'est la policy **`menu_items_public_beer_availability`** de [menu_items](./menu_items.md) (`SELECT` pour `authenticated`, prédicat `beer_id IS NOT NULL AND is_active`, exactement le `WHERE` de la vue) qui rend la disponibilité lisible. Le lint Supabase `security_definer_view` ne la signale plus.

`GRANT SELECT` à `authenticated` et `service_role`, **et rien d'autre**. `anon` a été retiré par la 105 : le front exige une session (le layout racine redirige vers le login) et la carte publique passe par [get_public_menu](../functions/get_public_menu.md). Aucun lecteur anonyme n'existait.

Corollaire : un compte connecté, client compris, peut lire directement les lignes « bière active » de `menu_items`, toutes colonnes. C'est du contenu de carte déjà public sur `menus.auxparaiges.fr`, plus la notion « disponible mais hors carte » que la vue exposait déjà.

### Historique : l'exception `security_definer` (096 à 105)

De la 096 à la 105, la vue était en `security_definer` (le défaut Postgres), à rebours de la convention du projet. Exception délibérée à l'époque : les tables `menu_*` sont fermées au public et la vue était le seul point par lequel la disponibilité restait lisible. Elle a été abandonnée parce que le lint `ERROR` qu'elle provoquait aurait fini par masquer une vraie erreur du même type. Deux leçons restent valables pour toute vue future :

> ⚠️ **Piège des privilèges par défaut.** La 096 accordait `SELECT` explicitement, mais Supabase pose des privilèges par défaut sur le schéma `public` qui donnent **tout** à `anon` et `authenticated` sur chaque nouvel objet : le `GRANT` s'y ajoutait au lieu de les restreindre. Or une vue simple sur une seule table est **auto-modifiable** par Postgres, et une vue `security_definer` contourne la RLS. Un `DELETE FROM beers_establishments WHERE id = X` émis avec la clé anon publique supprimait donc une ligne de `menu_items`. Corrigé par la **098** : `REVOKE ALL` puis `GRANT SELECT`. La règle : sur une vue, **révoquer avant d'accorder**. Un `GRANT` seul ne restreint rien.
>
> ⚠️ **Préférer `security_invoker` plus une policy dédiée** à une vue `security_definer`, même quand l'exposition est le but : le résultat est le même pour le lecteur, la RLS reste la seule autorité, et l'advisor Supabase reste vide.

## Exemples de requêtes

```sql
-- Bières disponibles dans un établissement (inchangé)
SELECT b.* FROM beers b
  JOIN beers_establishments be ON b.id = be.beer_id
 WHERE be.establishment_id = 5;

-- Établissements proposant une bière (inchangé)
SELECT e.* FROM establishments e
  JOIN beers_establishments be ON e.id = be.establishment_id
 WHERE be.beer_id = 959;

-- Nouveau : les bières disponibles mais absentes de la carte affichée
SELECT * FROM menu_items
 WHERE beer_id IS NOT NULL AND is_active AND category_id IS NULL;
```

## Statistiques

- 43 liaisons reprises au backfill de la migration 096, puis **95 disponibilités** après l'import des cartes (migration 099) : le Delirium passe de 3 à 54 bières, La Ripaille de 2 à 3 avec sa bière maison Saint-Martin.
