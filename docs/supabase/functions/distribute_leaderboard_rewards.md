# Function: distribute_leaderboard_rewards

Distribue automatiquement les récompenses aux TOP 10 du leaderboard.

> ⚠️ **Legacy — remplacée par [`distribute_period_rewards_v2`](./distribute_period_rewards_v2.md).** Barèmes en dur dans le corps de la fonction, `service_role` uniquement, appelée par aucun cron ni aucun des 4 projets applicatifs. Elle porte toujours le défaut corrigé par la migration 081 sur la v2 : elle lit les vues matérialisées `*_xp_leaderboard` (câblées sur `now()`) et résout sa période avec `get_period_identifier()`, donc un appel juste après une bascule de période travaille sur un classement vide. Ne pas l'appeler ; à supprimer ou à corriger séparément.

## Signature

```sql
CREATE FUNCTION distribute_leaderboard_rewards(
  p_period_type VARCHAR,
  p_force BOOLEAN DEFAULT false
) RETURNS JSON
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public'
```

## Paramètres

| Paramètre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `p_period_type` | `VARCHAR` | ✅ | - | Type: `weekly`, `monthly`, `yearly` |
| `p_force` | `BOOLEAN` | ❌ | `false` | Force la distribution même si déjà faite |

## Retour

### Succès

```json
{
  "success": true,
  "period_type": "weekly",
  "period_identifier": "2025-W03",
  "rewards_distributed": 10,
  "duration_ms": 1234,
  "errors": []
}
```

### Déjà distribué

```json
{
  "success": false,
  "message": "Rewards already distributed for period 2025-W03",
  "period_type": "weekly",
  "period_identifier": "2025-W03"
}
```

## Récompenses par Rang

### Hebdomadaire (weekly)

| Rang | Coupon Montant | Coupon % | Badge |
|------|----------------|----------|-------|
| 1er | 10,00€ | 15% | `weekly_champion` |
| 2-3 | 5,00€ | 10% | `weekly_podium_2_3` |
| 4-10 | 3,00€ | 5% | `weekly_top_10` |

### Mensuel (monthly)

| Rang | Coupon Montant | Coupon % | Badge |
|------|----------------|----------|-------|
| 1er | 40,00€ | 15% | `monthly_champion` |
| 2-3 | 20,00€ | 10% | `monthly_podium_2_3` |
| 4-10 | 12,00€ | 5% | `monthly_top_10` |

### Annuel (yearly)

| Rang | Coupon Montant | Coupon % | Badge |
|------|----------------|----------|-------|
| 1er | 480,00€ | 15% | `yearly_champion` |
| 2-3 | 240,00€ | 10% | `yearly_podium_2_3` |
| 4-10 | 144,00€ | 5% | `yearly_top_10` |

## Flux d'Exécution

```
1. Calcul de l'identifiant de période (get_period_identifier)
         │
         ▼
2. Vérification si déjà distribué (check_period_closed)
   └── Si oui et pas force → retourne erreur
         │
         ▼
3. Sélection de la vue leaderboard
   ├── weekly → weekly_xp_leaderboard
   ├── monthly → monthly_xp_leaderboard
   └── yearly → yearly_xp_leaderboard
         │
         ▼
4. Pour chaque utilisateur du TOP 10:
   ├── Déterminer le palier (1er, 2-3, 4-10)
   ├── Calculer les montants selon période
   ├── create_leaderboard_reward_coupon (montant)
   ├── create_leaderboard_reward_coupon (pourcentage)
   ├── award_user_badge
   └── INSERT leaderboard_reward_distributions
         │
         ▼
5. INSERT period_closures
         │
         ▼
6. Retour du résultat JSON
```

## Idempotence

La fonction est **idempotente** : si appelée plusieurs fois pour la même période, elle ne distribue pas les récompenses en double (sauf si `p_force = true`).

La vérification se fait via la table `period_closures`.

## Exemple d'Utilisation

```sql
-- Distribution hebdomadaire
SELECT distribute_leaderboard_rewards('weekly');

-- Distribution mensuelle
SELECT distribute_leaderboard_rewards('monthly');

-- Forcer une re-distribution
SELECT distribute_leaderboard_rewards('weekly', true);
```

### TypeScript

```typescript
// Appel via RPC (généralement appelé par un cron job)
const { data, error } = await supabase.rpc('distribute_leaderboard_rewards', {
  p_period_type: 'weekly',
  p_force: false
});

if (data?.success) {
  console.log(`${data.rewards_distributed} récompenses distribuées`);
  console.log(`Durée: ${data.duration_ms}ms`);
} else {
  console.log(data?.message || data?.error);
}
```

## Tables Impactées

| Table | Action |
|-------|--------|
| `coupons` | INSERT (2 par utilisateur) |
| `user_badges` | INSERT (1 par utilisateur) |
| `leaderboard_reward_distributions` | INSERT/UPDATE |
| `period_closures` | INSERT/UPDATE |

## Gestion des Erreurs

Les erreurs individuelles (par utilisateur) sont capturées et stockées dans `error_logs` mais n'arrêtent pas la distribution pour les autres utilisateurs.

```json
{
  "errors": [
    {
      "customer_id": "...",
      "rank": 5,
      "error": "Badge already awarded for this period"
    }
  ]
}
```

## Notes

- Doit être appelé APRÈS la fin de la période (ex: lundi matin pour la semaine précédente)
- Utilise les vues matérialisées donc les données doivent être à jour
- Les utilisateurs avec 0 XP ne sont pas dans le leaderboard
