# Function: distribute_quest_reward

Rejeu manuel (admin) de la distribution des récompenses d'une ligne `quest_progress`. **Depuis la migration 066 (quêtes répétables), c'est un simple « touch »** : la fonction déclenche le trigger `distribute_quest_rewards()` (source unique de vérité de toute distribution) via `UPDATE quest_progress SET updated_at = now()`, puis rapporte ce qui a été distribué.

Distinct du trigger automatique `distribute_quest_rewards()` (avec un `s`) qui s'exécute sur chaque UPDATE de `quest_progress` (+ trigger « touch » AFTER INSERT pour les lignes insérées directement complétées).

## Signature

```sql
CREATE FUNCTION public.distribute_quest_reward(p_quest_progress_id BIGINT)
RETURNS json
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
```

## Parametres

| Parametre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `p_quest_progress_id` | `BIGINT` | Oui | ID de la ligne `quest_progress` à (re)distribuer. |

> **Migration 040 (18/05/2026)** : le paramètre `p_admin_id` a été retiré ; signature inchangée depuis.

## Comportement (migration 066)

1. `PERFORM assert_admin()` — admin only.
2. Refuse une ligne inexistante (`Quest progress not found`) ou expirée (`Quest progress expired`).
3. Snapshot de `completions_count`, puis touch → le trigger décide : itérations nouvellement atteignables (cumul ≥ seuils cumulés, plafond niveau via [`get_max_quest_completions`](./get_max_quest_completions.md)), distribution par itération (coupon + bonus XP + bonus PdB en `gains`, badge 1×/période), une ligne `quest_completion_logs` par itération.
4. Si `completions_count` n'a pas bougé → `{"success": false, "error": "Nothing to distribute"}` (état déjà à jour — un skip, pas une erreur). Effet de bord utile : le touch normalise aussi le `status` de la ligne.
5. Sinon retourne le détail des itérations distribuées.

> ⚠️ **Changement de comportement 066** : l'ancienne implémentation parallèle créditait directement `profiles.total_xp` / `profiles.cashback_balance`. Le rejeu admin écrit désormais des lignes **`gains`** (`source_type = 'bonus_cashback_quest'`), comme le chemin automatique — correction de la divergence historique.

## Retour

```json
{
  "success": true,
  "quest_id": 35,
  "quest_name": "Défi des nectars",
  "customer_id": "uuid",
  "period_identifier": "2026-W29",
  "completions_count": 2,
  "status": "in_progress",
  "iterations_distributed": [
    { "iteration": 2, "coupon_id": null, "badge_id": null, "bonus_xp": 50, "bonus_cashback": 200 }
  ]
}
```

Cas d'échec (retourne `{ "success": false, "error": "..." }`) :
- `Quest progress not found`
- `Quest progress expired`
- `Nothing to distribute`

## Fonction batch : distribute_all_quest_rewards

```sql
distribute_all_quest_rewards(p_admin_id uuid DEFAULT NULL) RETURNS json
```

Parcourt les lignes `quest_progress` avec `status IN ('in_progress','completed') AND current_value >= target_value` (borne basse volontairement large : le trigger, idempotent, décide réellement) et appelle `distribute_quest_reward(id)` pour chacune. Retourne `{distributed, skipped, errors}` — un « Nothing to distribute » compte comme skip.

> Réparée par la 066 : elle appelait un overload 2-args `distribute_quest_reward(id, admin_id)` **inexistant** (cassée depuis la migration 040). Utile notamment pour payer les reliquats `completed` pré-066 (30 complétions légitimes jamais récompensées à cause du bug « complétion au premier INSERT », corrigé par la 066).

## Securite

- Première instruction : `PERFORM public.assert_admin()`. Voir [`assert_admin.md`](./assert_admin.md).
- `GRANT EXECUTE TO authenticated`, `REVOKE EXECUTE FROM PUBLIC, anon` (ACL préservées par `CREATE OR REPLACE`).
- L'audit trail est dérivé de `auth.uid()`.

## Notes

- Idempotente via le compteur monotone `quest_progress.completions_count` : appeler 2× ne distribue pas 2 fois.
- Plus besoin de repasser manuellement un statut à `completed` pour rejouer : le trigger recalcule tout depuis `current_value` / `completions_count`.

## Appelée par

- Dashboard admin (`questService.distributeQuestReward` / `distributeAllQuestRewards`).
