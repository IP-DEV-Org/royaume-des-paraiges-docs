# Vue: beers_establishments

> ⚠️ **Ce n'est plus une table.** Depuis la migration **096 (07/09/2026)**, `beers_establishments` est une **vue** sur [menu_items](./menu_items.md). L'ancienne table est conservée sous le nom `beers_establishments_legacy`, en lecture `service_role` uniquement, le temps de valider les applications ; elle ne reçoit plus aucune écriture et se périme donc à partir de cette date.

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

La vue est en **`security_definer`** (le défaut Postgres) et non `security_invoker`, à rebours de la convention posée par les migrations `security_*`.

C'est une **exception délibérée**. Ces migrations avaient forcé `security_invoker` sur des vues qui exposaient par ricochet des données d'administration. Ici, l'exposition est le but : les tables `menu_*` sont fermées au public, et cette vue est le seul point par lequel la disponibilité des bières reste lisible par le front et le dashboard, comme elle l'était avant. Son périmètre tient en une clause auditable, et elle n'expose que les cinq colonnes que la table exposait déjà.

`GRANT SELECT` à `anon`, `authenticated`, `service_role`.

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

- 43 liaisons reprises au backfill de la migration 096.
