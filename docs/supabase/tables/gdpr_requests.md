# Table: gdpr_requests

Suivi des demandes RGPD (export et suppression de données personnelles).

## Informations

| Propriete | Valeur |
|-----------|--------|
| **Schema** | `public` |
| **RLS** | Active |

## Colonnes

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | `uuid` | Non | gen_random_uuid() | Identifiant unique |
| `user_id` | `uuid` | Non | - | Utilisateur concerné (FK → profiles) |
| `request_type` | `text` | Non | - | Type de demande : `export` ou `erasure` |
| `status` | `text` | Non | 'pending' | Statut : `pending`, `completed` |
| `requested_at` | `timestamp with time zone` | Non | now() | Date de la demande |
| `processed_at` | `timestamp with time zone` | Oui | - | Date de traitement |
| `processed_by` | `uuid` | Oui | - | Admin ayant traité la demande (FK → profiles) |
| `notes` | `text` | Oui | - | Notes de traitement (détails JSON pour erasure) |
| `created_at` | `timestamp with time zone` | Non | now() | Date de création |

## Cles primaires

- `id`

## Relations (Foreign Keys)

| Colonne | Table cible | Colonne cible |
|---------|------------|---------------|
| `user_id` | `profiles` | `id` |
| `processed_by` | `profiles` | `id` |

## Politiques RLS

| Politique | Commande | Description |
|-----------|----------|-------------|
| `Users can create their own GDPR requests` | INSERT | `user_id = auth.uid()` |
| `Users can view their own GDPR requests` | SELECT | `user_id = auth.uid()` |
| `Admins can manage all GDPR requests` | ALL | Role admin uniquement |
| `Only admins can delete GDPR requests` | DELETE | Role admin uniquement |

## Fonctions associées

- `gdpr_export_user_data(target_user_id)` — Exporte toutes les données d'un utilisateur (insère une ligne `export` automatiquement)
- `gdpr_anonymize_user(target_user_id)` — Anonymise un utilisateur et insère une ligne `erasure`
- `gdpr_enforce_retention()` — Purge automatique des comptes inactifs > 3 ans
