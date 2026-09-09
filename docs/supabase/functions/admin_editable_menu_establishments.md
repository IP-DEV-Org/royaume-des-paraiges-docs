# Function: admin_editable_menu_establishments

Migration **113 (09/09/2026)**. Renvoie les identifiants des établissements dont l'appelant peut **modifier la carte** : la même règle que [admin_can_edit_menu](./admin_can_edit_menu.md), sous forme de liste, pour que le dashboard affiche exactement le périmètre que la RLS applique.

## Signature

```sql
CREATE FUNCTION public.admin_editable_menu_establishments()
RETURNS SETOF integer
LANGUAGE sql STABLE
SECURITY DEFINER
SET search_path = public
```

## Résultat

| Appelant | Lignes renvoyées |
|---|---|
| Super admin | Tous les établissements. |
| Admin avec la fonctionnalité `menus` et un rattachement | Son `attached_establishment_id` et les établissements du même `group_id` (non NULL). |
| Admin sans rattachement, admin privé de `menus`, autre rôle | Aucune. |

`SETOF integer` plutôt qu'un tableau : PostgREST renvoie les lignes telles quelles (`[1, 3, 4]`), sans cast côté client.

## Utilisée par

Le hook `useMenuAccess` du dashboard admin (`src/app/(dashboard)/menus/_lib/access.ts`, query key `menuKeys.editable()`, service `getEditableMenuEstablishments`). La règle n'est **pas** recalculée côté client : une seule source de vérité.

## Sécurité

`SECURITY DEFINER` (lit `profiles` et `establishments` sans dépendre de leurs policies), `REVOKE ALL FROM PUBLIC, anon`, `GRANT EXECUTE TO authenticated, service_role`.
