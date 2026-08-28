# Function: get_previous_period_identifier

Retourne l'identifiant de la période **close** : celle qui précède la période en cours. Fonction ancienne, réécrite par la migration **081c (28/08/2026)**.

## Pourquoi elle compte

Elle est le défaut de période de [`distribute_period_rewards_v2`](./distribute_period_rewards_v2.md) depuis la migration 081, et elle sert déjà à `get_email_report_payload` et `seed_email_report_last_period` pour résoudre la portée `previous` des rapports e-mail.

Les trois crons de distribution se déclenchent **juste après** la bascule de période — lundi 00:05, 1er du mois 00:10, 1er janvier 00:15, base en UTC — et appellent sans identifiant. Résoudre la période *en cours* à cet instant reviendrait à distribuer sur une période vieille de cinq minutes : c'est le défaut corrigé par la 081.

## Signature

```sql
CREATE FUNCTION public.get_previous_period_identifier(
  p_period_type CHARACTER VARYING
) RETURNS CHARACTER VARYING
LANGUAGE plpgsql
STABLE
SET search_path = public, pg_temp
```

⚠️ **Une seule signature, à un argument.** La migration 081 avait créé une surcharge `(varchar, timestamptz DEFAULT now())` sans voir la fonction existante : deux surcharges rendent ambigu tout appel à un argument (`42725 function ... is not unique`), ce qui a cassé `get_email_report_payload` et le défaut des crons de distribution. La 081b a supprimé la surcharge. **Ne pas la réintroduire.**

## Implémentation

```sql
SELECT pb.period_start INTO v_current_start
FROM get_period_bounds(p_period_type, get_period_identifier(p_period_type, now())) pb;

RETURN get_period_identifier(p_period_type, v_current_start - INTERVAL '1 second');
```

Une seconde avant l'ouverture de la période courante = dernier instant de la précédente, quelle que soit sa durée. Un `p_period_type` inconnu lève l'exception de `get_period_bounds`.

⚠️ **Ne jamais revenir à une soustraction d'intervalle.** L'implémentation d'origine faisait `now() - INTERVAL '7 days'` en hebdo et `date_trunc('month', now()) - INTERVAL '1 day'` en mensuel — deux raisonnements ad hoc, et **aucune branche `yearly`** (ni `ELSE` ni `RETURN` final), donc `get_previous_period_identifier('yearly')` levait `2F005 control reached end of function without RETURN`. C'est ce que le cron `distribute-yearly-rewards` aurait rencontré au 1er janvier.

## Équivalence vérifiée (081c)

Avant remplacement, ancienne logique contre nouvelle sur **2024 instants** couvrant 2024-01-01 → 2027-01-01 par pas de 13 h : **0 divergence** en hebdomadaire comme en mensuel. Les rapports e-mail ne changent donc pas de période. Seul `yearly` change de comportement : d'une exception à une valeur correcte.

## Valeurs vérifiées en production (28/08/2026, base en UTC)

| Appel | Retour |
|-------|--------|
| `get_previous_period_identifier('weekly')` | `2026-W34` |
| `get_previous_period_identifier('monthly')` | `2026-07` |
| `get_previous_period_identifier('yearly')` | `2025` |

Bascules validées par simulation avant déploiement : au 1er janvier 2026, l'hebdo renvoie `2025-W52` (et non `2026-W00`), le mensuel `2025-12`, l'annuel `2025`. Un cron rattrapé le mardi vise toujours la semaine close — c'est ce qui rend le défaut plus robuste qu'un identifiant figé dans la commande du cron.

## Voir aussi

- [`distribute_period_rewards_v2`](./distribute_period_rewards_v2.md) — défaut de période des 3 crons de distribution
- [`get_email_report_payload`](./get_email_report_payload.md) — portée `previous` des rapports e-mail
- [`get_period_leaderboard`](./get_period_leaderboard.md) — classement d'une période donnée
