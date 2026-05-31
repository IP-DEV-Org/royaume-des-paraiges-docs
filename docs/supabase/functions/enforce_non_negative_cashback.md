# Function / Trigger: enforce_non_negative_cashback

Garde-fou introduit par la migration **043 (31/05/2026)**. Empêche toute mutation de la table `gains` qui rendrait le solde de **Paraiges de Bronze (PdB)** d'un client négatif.

## Contexte

Les annulations de gains PdB se font par **insertion d'un gain négatif** (`source_type = 'rollback_beta_correction'`). Sans contrôle, une annulation peut dépasser le solde disponible et faire passer le client en négatif.

Incident (mai 2026) : 2 clients à `-476` et `-390` PdB après annulation de gains sans vérification. Recrédités manuellement via `credit_bonus_cashback`, puis ce garde-fou ajouté.

Solde PdB d'un client :

```
solde = SUM(gains.cashback_money)
      - SUM(receipt_lines.amount WHERE payment_method = 'cashback')
```

## Signature

```sql
CREATE FUNCTION public.enforce_non_negative_cashback() RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public

CREATE TRIGGER trg_enforce_non_negative_cashback
AFTER INSERT OR UPDATE OF cashback_money, customer_id OR DELETE ON public.gains
FOR EACH ROW EXECUTE FUNCTION public.enforce_non_negative_cashback();
```

## Logique

1. Détermine le client concerné (`OLD.customer_id` pour DELETE, sinon `NEW.customer_id`).
2. Recalcule le solde PdB **post-changement** (trigger `AFTER` : la table reflète déjà la mutation).
3. Si `solde < 0` → `RAISE EXCEPTION ... USING ERRCODE = 'P0423'` (message préfixé `CASHBACK_BALANCE_NEGATIVE:`), ce qui annule la transaction.
4. Cas particulier : un `UPDATE` qui change `customer_id` revérifie aussi l'**ancien** client.

## Portée (volontairement limitée)

- Couvre **uniquement** les mutations de `gains` : insertion d'un gain négatif, `DELETE`, `UPDATE` réducteur de `cashback_money`.
- Ne couvre **pas** le chemin de dépense (`receipt_lines` / `create_receipt`) : la dépense en PdB garde sa propre logique de contrôle côté scanner/waiters. Comme `create_receipt` insère un gain *positif* (gain `receipt`), le trigger ne le bloque jamais à tort.

## Erreur

- `SQLSTATE P0423`, message `CASHBACK_BALANCE_NEGATIVE: le solde de Paraiges de Bronze du client <uuid> deviendrait négatif (<n> PdB).`

## À savoir

- Le trigger est `AFTER` et `FOR EACH ROW` : il ne réévalue que le client touché par la ligne mutée, jamais l'historique existant (les gains négatifs déjà présents restent intacts).
- Pour un futur rollback de masse légitime qui devrait dépasser le solde, il faudra d'abord recréditer le client (ex. `credit_bonus_cashback`) ou désactiver temporairement le trigger — c'est intentionnel.
