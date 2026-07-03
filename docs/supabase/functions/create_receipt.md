# Function: create_receipt

Fonction principale pour créer un reçu avec paiements, coupons et gains.

## Signature

```sql
CREATE FUNCTION create_receipt(
  p_customer_id UUID,
  p_establishment_id BIGINT,
  p_payment_methods JSONB,
  p_coupon_ids BIGINT[] DEFAULT ARRAY[]::BIGINT[],
  p_employee_id UUID DEFAULT NULL,
  p_consumption_items JSONB DEFAULT NULL
) RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public'
```

## Paramètres

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `p_customer_id` | `UUID` | ✅ | ID du client (profiles.id) |
| `p_establishment_id` | `BIGINT` | ✅ | ID de l'établissement |
| `p_payment_methods` | `JSONB` | ✅ | Tableau des méthodes de paiement |
| `p_coupon_ids` | `BIGINT[]` | ❌ | IDs des coupons à utiliser |
| `p_employee_id` | `UUID` | ❌ | ID de l'employe createur. Si NULL, utilise auth.uid() |
| `p_consumption_items` | `JSONB` | ❌ | Tableau des types de consommation (optionnel) |

### Format de p_payment_methods

```json
[
  {"method": "card", "amount": 1500},
  {"method": "cash", "amount": 500},
  {"method": "cashback", "amount": 300}
]
```

| Méthode | Description | Génère des gains |
|---------|-------------|------------------|
| `card` | Carte bancaire | ✅ |
| `cash` | Espèces | ✅ |
| `cashback` | Solde cashback | ❌ |

> **Note** : Le type `coupon` n'est plus utilisé dans les receipt_lines. Les coupons pourcentage ajoutent un bonus cashback sans réduire le prix.

### Format de p_consumption_items

```json
[
  {"type": "biere", "quantity": 3},
  {"type": "cocktail", "quantity": 1}
]
```

| Type | Description |
|------|-------------|
| `cocktail` | Cocktails |
| `biere` | Bieres |
| `alcool` | Alcools (hors biere/cocktail) |
| `soft` | Boissons sans alcool |
| `boisson_chaude` | Cafe, the, chocolat, etc. |
| `restauration` | Nourriture |

> **Note** : Ce parametre est entierement optionnel. Si `NULL` ou tableau vide, aucun consumption item n'est cree.

## Retour

### Succès

```json
{
  "success": true,
  "receipt_id": 123,
  "total_amount": 2500,
  "payment_amount": 2500,
  "gains": {
    "gain_id": 456,
    "xp_base": 150,
    "xp_coefficient": 100,
    "xp_gained": 150,
    "cashback_base": 75,
    "cashback_coefficient": 100,
    "cashback_coupon_bonus": 375,
    "cashback_gained": 450,
    "amount_for_gains": 1500
  },
  "cashback_balance": {
    "available_before": 1000,
    "spent": 300,
    "earned": 450,
    "available_after": 1150
  },
  "coupons_used": 1
}
```

> **Note** : `cashback_coupon_bonus` est le bonus cashback additionnel généré par les coupons pourcentage. Il est calculé sur le montant total de la commande (`total_amount * coupon_percentage / 100`).
```

### Erreur

```json
{
  "success": false,
  "error": "Message d'erreur",
  "error_detail": "CODE_ERREUR"
}
```

## Étapes d'Exécution

1. **Vérification des permissions** - Seuls `employee`, `establishment`, `admin` peuvent créer (un `client` est rejeté : « Permissions insuffisantes »)
1b. **Scope établissement (migration 061)** - Un `employee` ou `establishment` ne peut créer un ticket que pour son `attached_establishment_id` : si `p_establishment_id` diffère → rejet ; si le compte n'a pas d'établissement rattaché → rejet. L'`admin` n'est pas contraint (multi-établissements). De plus, pour un `employee`, `p_employee_id` est **forcé à `auth.uid()`** (impossible d'attribuer le ticket à un autre employé). `anon` n'a plus le droit `EXECUTE`.
2. **Récupération des coefficients** - `xp_coefficient` et `cashback_coefficient` du profil
3. **Validation des coupons** - Via `validate_coupons()` (seuls les coupons % sont acceptés)
4. **Validation des paiements** - Via `validate_payment_methods()`
5. **Vérification du solde cashback** - Via `check_cashback_balance()`
6. **Calcul du montant total** (= montant des paiements, les coupons ne réduisent plus le prix)
7. **Création du receipt**
8. **Création des receipt_lines** (paiements uniquement)
8b. **Création des consumption items** (optionnel) - Si `p_consumption_items` est fourni, insere dans `receipt_consumption_items`
9. **Calcul des gains** - Via `calculate_gains()`
10. **Ajout du bonus cashback coupon** - Si coupon %, ajoute `total_amount * percentage / 100` au cashback
11. **Création du gain** (avec `customer_id` et `source_type='receipt'`)
12. **Mise à jour de la progression des quêtes** - Via `update_quest_progress_for_receipt()`
12b. **Attribution realtime des badges succès** - Via `award_achievements_for_user(p_customer_id, 'realtime')` (migration `026`, avril 2026). **Encapsulé dans un bloc `BEGIN...EXCEPTION WHEN OTHERS THEN RAISE WARNING`** : un échec ici ne casse JAMAIS la création du ticket (le handler global aurait sinon capturé l'erreur et renvoyé `{success: false}`).
13. **Marquage des coupons utilisés**
14. **Rafraîchissement des vues matérialisées**

> **Note** : La mise à jour des quêtes (étape 12) est appelée explicitement après la création des gains pour garantir que les quêtes de type `xp_earned` puissent calculer correctement la progression XP.
>
> **Note achievements** : l'étape 12b ne traite que les badges achievement en mode `realtime`. Les badges en mode `cron` (ex: `achievement_consecutive_weekly_quests_4`) sont évalués par le job pg_cron `award_achievements_cron` à 02:00 UTC. Cf. `functions/achievement_badges.md`.

## Exemple d'Utilisation

### TypeScript/JavaScript

```typescript
// Paiement simple en carte
const { data, error } = await supabase.rpc('create_receipt', {
  p_customer_id: customerId,
  p_establishment_id: 1,
  p_payment_methods: [
    { method: 'card', amount: 2500 }
  ]
});

// Paiement mixte avec cashback et coupon
const { data, error } = await supabase.rpc('create_receipt', {
  p_customer_id: customerId,
  p_establishment_id: 1,
  p_payment_methods: [
    { method: 'card', amount: 1500 },
    { method: 'cashback', amount: 500 }
  ],
  p_coupon_ids: [42]
});

if (data?.success) {
  console.log('XP gagnés:', data.gains.xp_gained);
  console.log('Cashback gagné:', data.gains.cashback_gained);
} else {
  console.error('Erreur:', data?.error);
}

// Paiement avec tracking des consommations
const { data, error } = await supabase.rpc('create_receipt', {
  p_customer_id: customerId,
  p_establishment_id: 1,
  p_payment_methods: [
    { method: 'card', amount: 3500 }
  ],
  p_consumption_items: [
    { type: 'biere', quantity: 3 },
    { type: 'cocktail', quantity: 1 }
  ]
});
```

### SQL Direct

```sql
SELECT create_receipt(
  p_customer_id := '123e4567-e89b-12d3-a456-426614174000'::UUID,
  p_establishment_id := 1,
  p_payment_methods := '[{"method": "card", "amount": 2500}]'::JSONB,
  p_coupon_ids := ARRAY[42]::BIGINT[]
);
```

## Triggers Déclenchés

Après l'insertion du receipt, deux triggers peuvent créer des coupons :

1. **trigger_weekly_coupon** - Si total semaine > 50€ → coupon 3,90€
2. **trigger_frequency_coupon** - Si ≥10 commandes semaine → coupon 5%

## Erreurs Possibles

| Erreur | Cause |
|--------|-------|
| `Permissions insuffisantes` | L'utilisateur n'est pas employee/admin/establishment |
| `Vous ne pouvez créer un ticket que pour votre établissement de rattachement` | `employee`/`establishment` avec un `p_establishment_id` ≠ `attached_establishment_id` (migration 061) |
| `Aucun établissement rattaché à ce compte` | `employee`/`establishment` sans `attached_establishment_id` (migration 061) |
| `Profil client non trouvé` | Le `customer_id` n'existe pas |
| `Coupon invalide` | Un coupon n'existe pas ou n'appartient pas au client |
| `Coupon déjà utilisé` | Un coupon a déjà été utilisé |
| `Solde cashback insuffisant` | Pas assez de cashback disponible |
| `Le montant total doit être positif` | Montant ≤ 0 |
| `Aucune méthode de paiement fournie` | Tableau de paiements vide |
| `La quantite de consommation doit etre un entier positif` | Quantite de consumption item ≤ 0 ou NULL |

## Système de Coupons (Bonus Cashback)

Depuis la migration vers le système de bonus cashback :

- **Coupons montant fixe** (ex: 5 EUR) : sont des **"Bonus Cashback"** crédités immédiatement au solde cashback du joueur dès l'attribution. Ils ne peuvent pas être utilisés sur une commande (`validate_coupons` les rejette).
- **Coupons pourcentage** (ex: 15%) : restent des **"Coupons"** mais au lieu de réduire le prix, ils ajoutent X% de cashback supplémentaire sur le montant total de la commande. Le client paie le montant total.
- La fonction `get_customer_available_coupons()` ne retourne plus que les coupons pourcentage.
- Il n'y a plus de `receipt_lines` avec `payment_method = 'coupon'`.

## Notes

- La fonction est **transactionnelle** : en cas d'erreur, tout est annulé
- Les vues matérialisées sont rafraîchies **CONCURRENTLY** (non-bloquant)
- Les gains de base sont calculés uniquement sur les paiements `card` et `cash`
- Le bonus cashback des coupons % est calculé sur le montant **total** de la commande
