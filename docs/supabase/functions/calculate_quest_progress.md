# Function: calculate_quest_progress

Calcule la progression actuelle d'un utilisateur pour une quête donnée et une période donnée. Supporte tous les `quest_type` de l'enum (`xp_earned`, `cashback_earned`, `amount_spent`, `establishments_visited`, `orders_count`, `quest_completed`, `consumption_count`).

Lisible par l'utilisateur sur sa propre progression ou par n'importe quel staff (`admin` / `employee` / `establishment`) depuis la migration **040 (Security Definer hardening, 18/05/2026)**.

## Signature

```sql
CREATE FUNCTION public.calculate_quest_progress(
  p_customer_id UUID,
  p_quest_id BIGINT,
  p_period_identifier VARCHAR DEFAULT NULL
) RETURNS integer
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
```

## Parametres

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `p_customer_id` | `UUID` | Oui | - | ID du client (`profiles.id`) |
| `p_quest_id` | `BIGINT` | Oui | - | ID de la quête. Lève une exception si la quête n'existe pas. |
| `p_period_identifier` | `VARCHAR` | Non | `NULL` | Identifiant de période (ex: `2026-W21`). Si `NULL`, fallback sur la période courante via `get_period_identifier(quest.period_type)`. |

## Retour

`INTEGER` : la progression courante, dans l'unité de la quête :

| `quest_type` | Unité |
|--------------|-------|
| `xp_earned` | XP cumulés sur la période (`SUM(gains.xp)` joint via `receipts`) |
| `cashback_earned` | Centimes / PdB cumulés (`SUM(gains.cashback_money)` joint via `receipts`) |
| `amount_spent` | Centimes dépensés (`SUM(receipts.amount)`) |
| `establishments_visited` | Nombre d'établissements distincts (déduplique par `group_id` si défini, sinon `id`) |
| `orders_count` | Nombre de receipts (`COUNT(*)`) |
| `quest_completed` | Nombre de sous-périodes distinctes avec au moins 1 quête complétée (`COUNT(DISTINCT period_identifier)` dans `quest_completion_logs`). Sous-période : weekly pour monthly, monthly pour yearly. Retourne 0 si `period_type = weekly` (pas de sous-période). |
| `consumption_count` | `SUM(receipt_consumption_items.quantity)` filtré sur `consumption_type = quest.consumption_type`. Lève une exception si `quests.consumption_type IS NULL`. |

## Logique

1. `PERFORM public.assert_self_or_staff(p_customer_id)` — voir Securite.
2. Charge la quête. Si introuvable → `RAISE EXCEPTION 'Quest not found: %'`.
3. Détermine `period_identifier` puis bornes via `get_period_bounds()`.
4. `CASE` sur `quest_type` : exécute la requête d'agrégation correspondante.
5. Retourne `v_progress`.

## Securite

Depuis la migration **040 (18/05/2026)** :

- Première instruction : `PERFORM public.assert_self_or_staff(p_customer_id)`. Voir [`assert_self_or_staff.md`](./assert_self_or_staff.md).
- Permet à un client de lire sa propre progression depuis l'app Expo, **et** à un employé / gérant / admin d'inspecter la progression de n'importe quel client (utile pour debug en salle ou via l'admin dashboard).
- Bypass automatique pour `service_role` et superuser (cas du cron qui réévalue la progression).

## Notes

- Cette fonction calcule à la demande — elle ne lit pas `quest_progress.current_value` (qui peut être obsolète). Pour la valeur stockée, lire directement la table `quest_progress`.
- Appelée par `update_quest_progress_for_receipt()` après chaque receipt pour rafraîchir `quest_progress.current_value`.
- Pour les meta-quêtes (`quest_completed`), assure que les sous-périodes possibles sont uniquement `monthly`→`weekly` et `yearly`→`monthly` (chaîne naturellement limitée).
