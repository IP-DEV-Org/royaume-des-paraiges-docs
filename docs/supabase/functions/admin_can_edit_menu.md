# Function: admin_can_edit_menu

Helper d'autorisation introduit par la migration **109 (08/09/2026)**, étendu par la **113 (09/09/2026)** : un admin ne modifie que la carte de son établissement de rattachement et celles des établissements de son **groupe** (`establishments.group_id`). Retourne `true` si l'appelant peut **écrire** sur la carte de l'établissement visé, `false` sinon. Ne concerne pas la lecture : tout admin lit toutes les cartes.

## Signature

```sql
CREATE FUNCTION public.admin_can_edit_menu(p_establishment_id integer)
RETURNS boolean
LANGUAGE sql STABLE
SECURITY DEFINER
SET search_path = public
```

## Paramètres

| Paramètre | Type | Description |
|---|---|---|
| `p_establishment_id` | integer | Établissement dont on veut modifier la carte. `NULL` rend `false` (sauf super admin). |

## Logique

```sql
SELECT public.admin_has_feature('menus')
   AND (
     public.is_super_admin()
     OR EXISTS (
       SELECT 1
         FROM public.profiles p
         JOIN public.establishments mine   ON mine.id   = p.attached_establishment_id
         JOIN public.establishments target ON target.id = p_establishment_id
        WHERE p.id = auth.uid()
          AND p.role = 'admin'
          AND (target.id = mine.id
               OR (mine.group_id IS NOT NULL AND target.group_id = mine.group_id))));
```

Quatre cas :

| Appelant | Résultat |
|---|---|
| Super admin (`is_super_admin()`) | `true` pour tout établissement. |
| Admin avec la fonctionnalité `menus`, rattaché à `p_establishment_id` | `true`. |
| Admin avec la fonctionnalité `menus`, rattaché à un établissement du **même groupe** (`group_id` non NULL, migration 113) | `true`. |
| Admin rattaché ailleurs ou hors groupe, admin sans rattachement, admin privé de la fonctionnalité `menus`, tout autre rôle | `false`. |

Exemple en production : le groupe « Les Paraiges (Metz) » réunit Aux Paraiges, le Garage et la Grange ; un admin rattaché à Aux Paraiges modifie les trois cartes.

`admin_has_feature('menus')` reste la première barrière : un super admin la passe toujours, un admin restreint par un super admin (table [admin_disabled_features](../tables/admin_disabled_features.md)) ne la passe jamais, rattaché ou non.

## Pourquoi le rattachement et le groupe, pas une table d'appartenances

Chaque gérant qui administre le dashboard est un compte `admin` déjà rattaché à son établissement par `profiles.attached_establishment_id`, et [establishment_groups](../tables/establishment_groups.md) modélise déjà « une même réalité métier » (caisse commune, comptoirs adjacents). Ces deux notions suffisent à décrire le périmètre ; une table d'appartenances aurait dupliqué l'information. Décision de la migration 095 conservée, groupe ajouté par la 113.

## Fonction sœur : `admin_editable_menu_establishments()`

Même règle, sous forme de liste (`SETOF integer`) : les identifiants des établissements dont l'appelant peut modifier la carte. C'est elle que lit le hook `useMenuAccess` du dashboard, pour que l'interface affiche exactement le périmètre que la base applique. Cf. [admin_editable_menu_establishments](./admin_editable_menu_establishments.md).

## Utilisée par

Les policies d'écriture (INSERT / UPDATE / DELETE) des tables `menu_*` qui portent une carte :

- **directement**, sur leur colonne `establishment_id` : [menu_categories](../tables/menu_categories.md), [menu_items](../tables/menu_items.md), [menu_sections](../tables/menu_sections.md), `menu_option_groups`, `menu_formulas`, `menu_establishment_events` ([satellites](../tables/menu_satellites.md)) ;
- **par remontée au parent** : [menu_item_variants](../tables/menu_item_variants.md) (via `menu_items`), `menu_options` (via `menu_option_groups`), `menu_formula_tiers` (via `menu_formulas`), `menu_item_option_groups` (les deux côtés, item **et** groupe, doivent être au même admin).

Les référentiels **partagés** entre cartes ([menu_item_types](../tables/menu_item_types.md), [menu_catalog_products](../tables/menu_catalog_products.md), `menu_events`) ne l'utilisent pas : leur écriture est réservée au super admin (`is_super_admin()`), puisque retitrer un soft du catalogue depuis un établissement modifierait les cartes des autres.

Le bucket `menu-assets` n'est pas scopé : ses objets ne portent pas d'établissement et aucun écran ne téléverse encore d'image.

## Effet sur PostgREST

Un `UPDATE` ou un `DELETE` hors périmètre ne lève **aucune erreur** : la RLS filtre les lignes atteignables, la requête « réussit » en touchant zéro ligne. Seuls l'`INSERT` et le `WITH CHECK` d'un `UPDATE` (déplacer une catégorie vers un autre établissement) lèvent `42501`. Deux compléments rendent ce silence audible :

- la RPC [set_menu_item_featured](../tables/menu_items.md) (SECURITY INVOKER) lève `42501 MENU_EDIT_FORBIDDEN` quand son `UPDATE` final ne trouve pas sa ligne ;
- le service admin `menuService.ts` relit les lignes touchées (`.select("id")`) et lève `42501 MENU_WRITE_IGNORED` s'il n'y en a pas.

## Sécurité

- `SECURITY DEFINER` : lit `profiles` avec les droits du owner, comme `admin_has_feature` ; la fonction ne dépend donc pas des policies de `profiles`.
- `REVOKE ALL FROM PUBLIC, anon`, `GRANT EXECUTE TO authenticated, service_role`. Seul `authenticated` évalue les policies qui l'appellent.
- Jamais d'exception : conçue pour les clauses `USING` / `WITH CHECK`.

## Côté admin

Le hook `useMenuAccess(establishmentId)` (`src/app/(dashboard)/menus/_lib/access.ts`) lit le périmètre via la RPC `admin_editable_menu_establishments` (query key `menuKeys.editable()`) pour ne pas proposer des gestes qui échoueraient : `/menus/[id]` passe en lecture seule (bandeau, lignes et en-têtes sans geste, barre d'actions retirée), les formulaires produit sont barrés par `MenuEditGuard`. La liste `/menus` met l'établissement de rattachement en tête, puis ceux de son groupe, et badge les autres « lecture seule ».

## Voir aussi

- [`is_super_admin`](./is_super_admin.md)
- Migration 070 (`admin_has_feature`, feature gating en RLS des quêtes), dont la 109 reprend le patron.
