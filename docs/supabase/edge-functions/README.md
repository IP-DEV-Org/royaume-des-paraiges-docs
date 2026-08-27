# Edge Functions

## Liste des Edge Functions (4)

| Slug | Nom | JWT Required | Status | Version |
|------|-----|--------------|--------|---------|
| `cashpad-webhook` | cashpad-webhook | Oui | ACTIVE | 2 |
| `cashpad-process-queue` | cashpad-process-queue | Oui | ACTIVE | 2 |
| `cashpad-reconcile-daily` | cashpad-reconcile-daily | Oui | ACTIVE | 13 |
| `send-email-reports` | send-email-reports | **Non** (à désactiver) | ACTIVE | 2 |

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

### send-email-reports

Envoie les **rapports e-mail automatisés** (migrations 076/077/078, étendues par 079/080) à leurs destinataires internes, via **Resend**. Les chiffres ne sont pas calculés ici : ils viennent de la RPC `get_email_report_payload`. La fonction résout quoi envoyer, rend le HTML et poste.

Trois gabarits de rendu dans `templates.ts`, un par `report_type` : `activity_summary`, `leaderboard`, `new_quests` (défis de la période). Ajouter un type de rapport sans ajouter sa branche dans `renderReport()` fait échouer l'envoi avec « Type de rapport non géré ».

- **Entrypoint** : `supabase/functions/send-email-reports/index.ts` (+ `templates.ts` pour le rendu)
- **Tables cibles** : `email_reports`, `email_report_recipients`, `email_report_runs`
- **Déclenchement** : cron `email-reports-dispatch` (07:00 UTC quotidien) ou appel depuis la page admin `/reports`

#### Modes (corps de la requête)

| Corps | Effet |
|---|---|
| `{}` | Passage cron : tous les rapports actifs dont la période visée n'a pas encore été envoyée |
| `{ "report_key": "..." }` | Envoi manuel d'un rapport, période visée par défaut |
| `{ "report_key": "...", "period_identifier": "2026-07" }` | Renvoi d'une période passée |
| `{ "report_key": "...", "preview": true }` | Rend le HTML **sans rien envoyer** (prévisualisation admin) |
| `{ "report_key": "...", "test_email": "..." }` | Envoi de test à une seule adresse ; **ne consomme pas la période** (`last_period_sent` inchangé) |

#### Idempotence

La période visée d'un rapport dépend de sa **portée** (`email_reports.period_scope`, migration 079) : période écoulée pour un bilan, période en cours pour une annonce (« les défis de la semaine »). C'est la RPC qui tranche — la fonction ne fait que comparer la période obtenue à `last_period_sent`.

Le cron passe **tous les jours** ; la période visée ne change qu'au changement de semaine ou de mois, et ce dans les deux portées : un rapport `current` bascule le lundi matin comme un rapport `previous`. Un rapport dont `last_period_sent` vaut déjà la période cible est `skipped`. Corollaire utile : un passage en échec est **rattrapé le lendemain**, au lieu d'attendre une semaine ou un mois. Un envoi manuel, lui, est un acte volontaire et peut renvoyer la même période.

#### Auth

`verify_jwt` doit être **désactivé** sur cette fonction, comme pour les fonctions Cashpad : les clés `sb_secret_…` ne sont pas des JWT et seraient rejetées par le gateway. La validation est faite intégralement dans le code, qui accepte :

1. la secret key Supabase (`EMAIL_REPORTS_SERVICE_KEY`, sinon `RECONCILE_SERVICE_KEY`, sinon `SUPABASE_SERVICE_ROLE_KEY`) : appel du cron ;
2. un JWT d'utilisateur dont `profiles.role = 'admin'` : appel depuis la page `/reports`.

#### Secrets requis

| Secret | Rôle |
|---|---|
| `RESEND_API_KEY` | Clé API Resend. Sans elle, la fonction répond `error` sans rien envoyer. |
| `EMAIL_REPORTS_FROM` | Expéditeur, en prod `Royaume des Paraiges <no-reply@mail-royaume.auxparaiges.fr>`. Le domaine doit être vérifié chez Resend (SPF/DKIM). |
| `EMAIL_REPORTS_REPLY_TO` | Adresse de réponse (optionnel) |
| `EMAIL_REPORTS_SERVICE_KEY` | Bearer attendu du cron (optionnel, fallback sur `reconcile_service_key`) |

#### Rendu

Les gabarits (`templates.ts`) reprennent le **design du dashboard admin** (shadcn/ui, thème clair) : cartes de chiffres à bord 1px et radius 8px, tableaux sans zébrures, typographie sans-serif, variations en émeraude/rouge. Les couleurs sont la conversion en hex des tokens `:root` de `src/app/globals.css` côté admin ; si la palette du dashboard bouge, la reporter dans `templates.ts`.

Contraintes de rendu e-mail : mise en page en `<table>`, styles inline uniquement, thème clair figé (un e-mail ne suit pas le dark mode), et aucune ressource externe. Chaque rapport est envoyé en HTML **et** en texte brut.

#### Envoi

Un e-mail **par destinataire** (et non un envoi groupé) pour tracer les échecs adresse par adresse, espacés de 550 ms pour rester sous la limite Resend de 2 requêtes/seconde. Budget de 120 s avant de renvoyer un résumé partiel. Chaque tentative est journalisée dans `email_report_runs`, avec un snapshot du payload.
