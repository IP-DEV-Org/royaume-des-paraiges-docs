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
- **Auth**: `verify_jwt=false` (gateway) + validation interne = **secret key Supabase** `sb_secret_…` (cron, fallback `SUPABASE_SERVICE_ROLE_KEY`) OU JWT admin (UI). Le cron envoie la clé depuis Vault (`reconcile_service_key`). `cashpad-process-queue` suit le même modèle (cron-only, sans branche JWT). ⚠️ À chaque **rotation** de la clé Supabase, mettre à jour le secret Vault sinon les crons retombent en 401.
- **Tables cibles**: `cashpad_receipts_snapshot`, `cashpad_reconciliations`, `cashpad_matching_params`

#### Pipeline

1. Charge `cashpad_matching_params` en mémoire
2. Pour chaque jour `D` de la plage et chaque establishment Cashpad — **par journée fiscale (clôture → clôture)** :
   - Sélectionne les **clôtures dont le service a OUVERT le jour D** (`range_begin_date ∈ [D, D+1)`) via `getArchivesOpenedOnDate` — **pas** les archives qui intersectent le jour calendaire. Une clôture est traitée d'un bloc (on ne coupe plus le service à minuit).
   - Fetch du contenu → upsert **tous** les tickets (annulés + 0€ inclus) dans `cashpad_receipts_snapshot`
   - Calcule la **fenêtre fiscale** = union des `[range_begin, range_end]` des archives retenues
   - Liste les receipts Royaume dont `created_at` ∈ fenêtre fiscale (± marge 300s), et non du jour calendaire
   - Pour chaque receipt : cashback exclusion → match contre les tickets **encaissés** (non annulés ET montant > 0) avec offset/window → fallback cancelled detection → score
   - Upsert dans `cashpad_reconciliations`. Si aucune clôture n'a ouvert le jour D → skip.
3. Garde-fou temps 120s → renvoie un summary partiel avec `next_start_date` si dépassement

> Le summary expose `cashpad_tickets_fetched` (tous les tickets snapshotés) **et** `cashpad_tickets_encaisses` (non annulé + montant > 0, = décompte affiché par Cashpad).

#### Fonctions clés internes

- `getArchivesOpenedOnDate()` (`_shared/cashpad-client.ts`) — sélectionne les clôtures par date d'**ouverture** du service (journée fiscale). `getArchivesForDate` (intersection calendaire) reste dispo mais n'est plus utilisée par la réconciliation.
- `fetchSnapshotsForTarget()` — fetch + snapshot d'une journée fiscale ; renvoie aussi `[windowStart, windowEnd]`.
- `matchReceipts()` — Logique de matching (montant strict, ticket **encaissé** uniquement, fenêtre adaptative, départage par employee mapping)
- `computeConfidence()` — Score de confiance (70% proximité temporelle + 30% unicité candidat)

Voir la section « Réconciliation Cashpad » du CLAUDE.md workspace pour le détail des règles de matching.
