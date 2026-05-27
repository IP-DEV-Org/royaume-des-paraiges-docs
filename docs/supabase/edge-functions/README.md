# Edge Functions

## Liste des Edge Functions (3)

| Slug | Nom | JWT Required | Status | Version |
|------|-----|--------------|--------|---------|
| `cashpad-webhook` | cashpad-webhook | Oui | ACTIVE | 2 |
| `cashpad-process-queue` | cashpad-process-queue | Oui | ACTIVE | 2 |
| `cashpad-reconcile-daily` | cashpad-reconcile-daily | Oui | ACTIVE | 13 |

> **Note** : L'edge function `send-contact-email` (anciennement documentée ici) a été supprimée lors de la migration vers le projet IPDEV.

## Details

### cashpad-webhook

Reçoit les notifications webhook de Cashpad lors de la création ou modification d'un ticket de caisse. Insère l'événement dans `cashpad_webhook_queue` pour traitement asynchrone (les archives Cashpad ne sont pas immédiatement disponibles après la notification).

- **Entrypoint**: `supabase/functions/cashpad-webhook/index.ts`
- **Auth**: JWT requis (service_role ou admin)
- **Table cible**: `cashpad_webhook_queue`

### cashpad-process-queue

Traite les éléments en attente dans `cashpad_webhook_queue`. Pour chaque événement :
1. Fetch les données du ticket depuis l'API d'archives Cashpad (`salesdata/v2`)
2. Upsert dans `cashpad_receipts_snapshot`
3. Déclenche le matching/réconciliation si un receipt Royaume correspondant existe

- **Entrypoint**: `supabase/functions/cashpad-process-queue/index.ts`
- **Auth**: JWT requis (service_role ou admin)
- **Tables cibles**: `cashpad_webhook_queue`, `cashpad_receipts_snapshot`, `cashpad_reconciliations`

### cashpad-reconcile-daily

Réconciliation quotidienne batch entre les receipts Royaume et les tickets Cashpad. Peut être déclenchée par cron (03:00 UTC) ou manuellement via l'UI admin.

- **Entrypoint**: `supabase/functions/cashpad-reconcile-daily/index.ts`
- **Auth**: service_role_key (cron) OU JWT admin (UI)
- **Tables cibles**: `cashpad_receipts_snapshot`, `cashpad_reconciliations`, `cashpad_matching_params`

#### Pipeline

1. Charge `cashpad_matching_params` en mémoire
2. Pour chaque jour de la plage et chaque establishment Cashpad :
   - Fetch des archives Cashpad → upsert dans `cashpad_receipts_snapshot`
   - Liste les receipts Royaume du jour
   - Pour chaque receipt : cashback exclusion → match avec offset/window → fallback cancelled detection → score
   - Upsert dans `cashpad_reconciliations`
3. Garde-fou temps 120s → renvoie un summary partiel avec `next_start_date` si dépassement

#### Fonctions clés internes

- `matchReceipts()` — Logique de matching (montant strict, fenêtre adaptative, départage par employee mapping)
- `computeConfidence()` — Score de confiance (70% proximité temporelle + 30% unicité candidat)

Voir la section « Réconciliation Cashpad » du CLAUDE.md workspace pour le détail des règles de matching.
