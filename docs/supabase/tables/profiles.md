# Table: profiles

Profils utilisateurs du Royaume des Paraiges. Synchronisé depuis `auth.users` via trigger `handle_new_user`. Rôles : `admin`, `establishment`, `employee`, `client`.

## Informations

| Propriete | Valeur |
|-----------|--------|
| **Schema** | `public` |
| **RLS** | Active |

## Colonnes

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | `uuid` | Non | gen_random_uuid() | Identifiant unique (= auth.users.id) |
| `created_at` | `timestamp with time zone` | Non | now() | Date de création |
| `attached_establishment_id` | `integer` | Oui | - | Établissement de rattachement (employees/establishment managers) |
| `email` | `text` | Oui | - | Email (synchronisé depuis auth.users) |
| `first_name` | `text` | Oui | - | Prénom |
| `last_name` | `text` | Oui | - | Nom de famille |
| `avatar_url` | `text` | Oui | - | URL de la photo de profil (bucket avatars) |
| `phone` | `text` | Oui | - | Numéro de téléphone |
| `birthdate` | `date` | Oui | - | Date de naissance |
| `updated_at` | `timestamp with time zone` | Oui | now() | Date de dernière modification |
| `username` | `text` | Oui | - | Nom d'utilisateur unique |
| `role` | `user_role` | Non | 'client'::user_role | Rôle : admin, establishment, employee, client |
| `xp_coefficient` | `integer` | Non | 100 | Multiplicateur XP (100 = ×1.0) |
| `cashback_coefficient` | `integer` | Non | 100 | Multiplicateur cashback (100 = ×1.0) |
| `is_test` | `boolean` | Non | false | Comptes test exclus des classements et distributions de récompenses |
| `deleted_at` | `timestamp with time zone` | Oui | - | Date de suppression (soft-delete RGPD, via `gdpr_anonymize_user`) |
| `cashpad_customer_id` | `text` | Oui | - | Identifiant client côté Cashpad |
| `age_certified_at` | `timestamp with time zone` | Oui | - | Date de certification de majorité |
| `terms_accepted_at` | `timestamp with time zone` | Oui | - | Date d'acceptation des CGU |
| `identity_photo_url` | `text` | Oui | - | URL de la photo d'identité (bucket avatars) |
| `identity_photo_updated_at` | `timestamp with time zone` | Oui | - | Date du dernier upload de photo d'identité (cooldown 30 jours) |

## Cles primaires

- `id`

## Relations (Foreign Keys)

| Colonne | Table cible | Colonne cible |
|---------|------------|---------------|
| `attached_establishment_id` | `establishments` | `id` |

## Triggers

| Trigger | Événement | Description |
|---------|-----------|-------------|
| `set_profiles_updated_at` | BEFORE UPDATE | Met à jour `updated_at` automatiquement |
| `trg_identity_photo_cooldown` | BEFORE UPDATE | Applique le cooldown de 30 jours sur le changement de photo d'identité |
