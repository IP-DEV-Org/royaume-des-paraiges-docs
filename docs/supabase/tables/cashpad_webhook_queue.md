# Table: cashpad_webhook_queue

Queue des événements webhook Cashpad en attente de traitement. Les tickets Cashpad ne sont pas immédiatement disponibles dans l'API d'archives après leur création — cette queue stocke les notifications en attendant que les données soient accessibles.

## Informations

| Propriete | Valeur |
|-----------|--------|
| **Schema** | `public` |
| **RLS** | Active |

## Colonnes

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | `bigint` | Non | - | Identifiant unique |
| `created_at` | `timestamp with time zone` | Non | now() | Date de réception du webhook |
| `installation_id` | `text` | Non | - | ID installation Cashpad source |
| `cashpad_receipt_id` | `text` | Non | - | ID du ticket Cashpad (UNIQUE) |
| `cashpad_sequential_id` | `integer` | Oui | - | Numéro séquentiel du ticket |
| `event_type` | `integer` | Non | 1 | Type d'événement webhook |
| `status` | `text` | Non | 'pending' | Statut : `pending`, `processed`, `failed` |
| `processed_at` | `timestamp with time zone` | Oui | - | Date de traitement |
| `error_message` | `text` | Oui | - | Message d'erreur si échec |
| `retry_count` | `integer` | Non | 0 | Nombre de tentatives de traitement |

## Cles primaires

- `id`

## Contraintes

| Contrainte | Type | Colonne(s) |
|-----------|------|------------|
| `uq_cashpad_webhook_queue_receipt_id` | UNIQUE | `cashpad_receipt_id` |

## Politiques RLS

| Politique | Commande | Description |
|-----------|----------|-------------|
| `cashpad_webhook_queue_admin_select` | SELECT | Role admin uniquement |

## Edge Functions associées

- `cashpad-webhook` — Reçoit les notifications et insère dans la queue
- `cashpad-process-queue` — Traite les éléments en attente (fetch archives + upsert snapshot + réconciliation)
