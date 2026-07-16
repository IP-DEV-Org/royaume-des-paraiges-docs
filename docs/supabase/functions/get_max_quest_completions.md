# Fonction: get_max_quest_completions

Nombre maximum de complétions d'un défi répétable par période pour un joueur donné, selon son **niveau de saison** et le barème admin. Introduite par la migration 066 (quêtes répétables).

## Signatures

```sql
get_max_quest_completions(p_customer_id uuid) RETURNS integer  -- interne + service_role
get_my_max_quest_completions() RETURNS integer                 -- wrapper client (auth.uid())
```

`STABLE`, `SECURITY DEFINER`, `search_path = public, pg_temp`.

## Logique (migration 068 — bimodale)

1. Calcule le niveau de saison : `compute_level_from_xp(get_season_xp(p_customer_id))`.
2. Lit `admin_settings['quest_repeat_level_tiers']` :
   - **Tableau non vide** `[{min_level, max_completions}]` → barème **manuel** : palier de plus haut `min_level <= niveau` (robuste même si non trié ; entrées mal typées ignorées ; aucun palier atteint → 1).
   - **`"auto"`, clé absente ou valeur invalide** → mode **automatique lié aux rangs** (défaut produit depuis la 068) : plafond = `MAX(ranks.sort_order)` des rangs dont `min_level <= niveau`. Rang 1 (Écuyer, niv. 1-5) → 1, rang 2 (Soldat, 6-10) → 2, … rang 6 (Chevalier de la Table Ronde) → 6. Le lien est **dynamique** : toute édition de la table `ranks` (admin `/content/storytelling`) est immédiatement répercutée.
3. **Garde-fou ultime** : toute exception (`EXCEPTION WHEN OTHERS`) → 1 (comportement historique, une seule complétion).

## Droits

| Fonction | anon | authenticated | service_role |
|---|---|---|---|
| `get_max_quest_completions(uuid)` | ✗ | ✗ | ✓ |
| `get_my_max_quest_completions()` | ✗ | ✓ | ✓ |

La version paramétrée n'est jamais exposée aux clients (pattern wrappers `security_user_stats_rpc_wrappers` de mai 2026).

## Consommateurs

- **Trigger `distribute_quest_rewards`** — plafond d'itérations, réévalué à **chaque** passage (un level-up en cours de période ré-ouvre une ligne `rewarded`). ⚠️ Depuis la migration 069, le trigger borne ce plafond par les itérations réellement configurées : `v_allowed = LEAST(get_max_quest_completions(...), 1 + COUNT(quest_iterations))`.
- **Front Expo** — `QuestService.getMyMaxQuestCompletions()` (RPC `get_my_max_quest_completions`), un appel par assemblage de la liste des défis, pour afficher « itération 2/3 ».

## Fonction sœur : get_quest_iteration_target

```sql
get_quest_iteration_target(p_quest_id bigint, p_iteration integer) RETURNS integer
```

Objectif d'une itération donnée : `COALESCE(quest_iterations.target_value, quests.target_value)`. `STABLE`, invoker rights, EXECUTE `authenticated` + `service_role`. Utilisée par le trigger pour construire les seuils cumulés.
