# Function: admin_delete_receipt

RPC introduite par la migration **044 (31/05/2026)**. Supprime un ticket (`receipts`) et **toute sa cascade**, depuis le dashboard admin (`/users/[id]` → onglet « Tickets de l'utilisateur »).

## Signature

```sql
CREATE FUNCTION public.admin_delete_receipt(p_receipt_id bigint) RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
```

`EXECUTE` accordé à `authenticated` et `service_role` ; révoqué pour `anon` / `PUBLIC`. Contrôle d'accès via `assert_admin()` en première instruction.

## Cascade BDD

Chaque table enfant de `receipts` est en `ON DELETE CASCADE` :

| Table enfant | FK |
|---|---|
| `receipt_lines` | `receipt_id` |
| `gains` | `receipt_id` |
| `receipt_consumption_items` | `receipt_id` |
| `spendings` | `receipt_id` |
| `cashpad_reconciliations` | `receipt_id` |

## Pourquoi une RPC plutôt qu'un simple `DELETE FROM receipts`

Le trigger `AFTER DELETE` **`trg_enforce_non_negative_cashback`** (migration 043) sur `gains` recalcule le solde PdB =
`SUM(gains.cashback_money) − SUM(receipt_lines.amount WHERE payment_method='cashback')`.

Lors d'un `DELETE` direct sur `receipts`, l'ordre de cascade entre `gains` et `receipt_lines` n'est **pas garanti**. Si les `gains` (PdB gagnés) du ticket sont purgés **avant** ses `receipt_lines` (PdB dépensés), le solde paraît temporairement négatif → faux positif `P0423` qui annule toute la transaction.

La RPC supprime donc dans un **ordre maîtrisé**, en une seule transaction atomique :

1. `DELETE FROM receipt_lines` — retire d'abord la contribution « dépense » du solde
2. `DELETE FROM gains` — le trigger voit alors le solde **final**, correct
3. `DELETE FROM receipts` — cascade le reste (`spendings`, `receipt_consumption_items`, `cashpad_reconciliations`)

Puis `REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats` pour que le solde / XP / nombre de tickets agrégés (admin + front) soient immédiatement à jour.

## Comportement quand le solde deviendrait réellement négatif

Si les PdB gagnés sur ce ticket ont **déjà été dépensés ailleurs**, supprimer les gains ferait passer le solde du client en négatif : le trigger `trg_enforce_non_negative_cashback` lève `P0423` et **toute** la suppression est annulée (transaction atomique). C'est intentionnel — on refuse de corrompre le solde. L'admin doit d'abord régulariser (ex. recréditer via `credit_bonus_cashback` ou annuler la dépense liée).

## Erreurs

| SQLSTATE | Cause | Message UI admin |
|---|---|---|
| `42501` | Appelant non-admin | (interne) |
| `P0002` | Ticket introuvable | « Impossible de supprimer ce ticket. » |
| `P0423` | Solde deviendrait négatif (PdB déjà dépensés) | « Les Paraiges de Bronze gagnés sur ce ticket ont déjà été dépensés ailleurs… » |

## À savoir

- Côté admin : `receiptService.deleteReceipt(id)` (valide l'input via `deleteReceiptSchema`, Zod) appelle la RPC. UI : bouton corbeille + `AlertDialog` de confirmation sur chaque ligne de la table tickets de `/users/[id]`.
- La suppression **recalcule** le `cashback_coefficient` (trigger sur `gains`) et donc potentiellement le niveau dérivé du XP de saison.
- `quest_progress` n'est **pas** recalculé par cette suppression (la progression est remise à zéro à chaque période, donc l'impact est borné).
