# Function: get_user_quests

Récupère toutes les quêtes actives avec la progression de l'utilisateur ciblé pour la période courante. Utilisée uniquement depuis l'admin (visualisation détail utilisateur).

Admin-only depuis la migration **040 (Security Definer hardening, 18/05/2026)**.

## Signature

```sql
CREATE FUNCTION public.get_user_quests(
  p_customer_id UUID,
  p_period_type VARCHAR DEFAULT NULL
) RETURNS json
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
```

## Parametres

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `p_customer_id` | `UUID` | Oui | - | ID du client (`profiles.id`) |
| `p_period_type` | `VARCHAR` | Non | `NULL` | Filtre optionnel : `'weekly'` / `'monthly'` / `'yearly'`. `NULL` = toutes les périodes. |

## Retour

`json` : tableau de quêtes avec progression jointe pour la période courante. Vide (`[]`) s'il n'y a pas de quête active.

Chaque ligne contient :

| Champ | Description |
|-------|-------------|
| `id`, `name`, `description`, `slug`, `quest_type`, `target_value`, `period_type`, `bonus_xp`, `bonus_cashback`, `display_order` | Champs de la quête |
| `coupon_template` | Objet `{ id, name, amount, percentage }` du template récompense (NULL si pas de template) |
| `badge_type` | Objet `{ id, name, slug, icon, rarity }` du badge récompense (NULL si pas de badge) |
| `current_value` | Progression courante (`COALESCE(quest_progress.current_value, 0)`) |
| `status` | `'not_started'` (défaut) / `'in_progress'` / `'completed'` / `'rewarded'` / `'expired'` |
| `completed_at`, `rewarded_at` | Timestamps de la progression |
| `current_period` | Identifiant de période courante calculé par `get_period_identifier(period_type)` |

## Logique

1. `PERFORM public.assert_admin()` — bloque les non-admins.
2. SELECT sur `quests` (filtre `is_active = true` + filtre optionnel `period_type = p_period_type`).
3. LEFT JOIN `coupon_templates` (récompense coupon).
4. LEFT JOIN `badge_types` (récompense badge).
5. LEFT JOIN `quest_progress` sur `quest_id` + `customer_id = p_customer_id` + `period_identifier = get_period_identifier(q.period_type)` (progression de la période courante uniquement).
6. Aggrège en `json` triée par `display_order`.

## Securite

Depuis la migration **040 (18/05/2026)** :

- Première instruction : `PERFORM public.assert_admin()`. Voir [`assert_admin.md`](./assert_admin.md).
- Utilisée uniquement depuis l'admin (`questService.getUserQuests` côté admin Next.js) — le front client (Expo) lit sa propre progression différemment.
- Aucun REVOKE supplémentaire imposé par la migration — la sûreté repose sur le guard.

## Notes

- Pour la progression historique (toutes périodes confondues, pas seulement la courante), utiliser `getUserQuestProgress` côté admin service qui lit directement `quest_progress` (avec join `quests`).
- La progression d'une quête `cashback_earned` est en PdB (1 PdB = 1 centime).
