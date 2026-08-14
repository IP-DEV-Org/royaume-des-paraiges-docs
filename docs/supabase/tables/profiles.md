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
| `is_super_admin` | `boolean` | Non | false | Super admin du dashboard : peut gérer les accès par fonctionnalité des autres admins (migration 057). Immuable hors `service_role`/`postgres` (trigger anti-escalade). |

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
| `trg_protect_is_super_admin` | BEFORE UPDATE | Rend `is_super_admin` immuable côté client (seuls `service_role`/`postgres`/`supabase_admin` peuvent le changer). Migration 057. |
| `trg_protect_staff_profile_delete` | BEFORE DELETE | Refuse la suppression d'un profil dont le `role` n'est pas `client` (`P0424 ACCOUNT_ROLE_PROTECTED`). Jumeau du trigger posé sur `auth.users` par la même migration **073** : les deux appellent `protect_staff_account_delete()`. Seul un accès SQL direct (`session_user` `postgres`/`supabase_admin`) est exempté ; `service_role` ne l'est **pas**, contrairement aux deux triggers ci-dessus. Pour supprimer un compte du personnel : le repasser en `client` d'abord. |
| `trg_protect_role` | BEFORE UPDATE | Restreint la modification de `role`. Un **super-admin** (`is_super_admin()`) — ou `service_role`/`postgres`/`supabase_admin` — peut attribuer n'importe quel rôle. Un **admin non super** peut uniquement basculer un compte entre `client` et `employee` (les deux sens) : `admin` et `establishment` restent réservés au super-admin, et un admin ne peut ni se rétrograder ni toucher un autre admin (`OLD.role` doit déjà être `client`/`employee`). Toute autre transition lève `42501 ROLE_CHANGE_FORBIDDEN`. Ne se déclenche que si `NEW.role IS DISTINCT FROM OLD.role`, donc un admin normal peut toujours éditer les autres champs d'un profil. ⚠️ Le test sur le rôle de l'**appelant** (`get_current_user_role() = 'admin'`) est load-bearing : la policy RLS self-update laisse chacun écrire sa propre ligne, sans lui un client se promouvrait `employee` (accès scanner/waiters) via `PATCH /rest/v1/profiles`. Migration 059 (créé, niveau admin) → 060 (resserré au super-admin) → 064 (rouvert à la bascule client ↔ employee pour les admins). |

## Contraintes CHECK

| Contrainte | Règle | Description |
|---|---|---|
| `profiles_staff_requires_establishment` | `role NOT IN ('employee', 'establishment') OR attached_establishment_id IS NOT NULL` | Tout compte du **personnel** (`employee` ou gérant `establishment`) doit être rattaché à un établissement — sans rattachement il n'a aucun périmètre (scanner/waiters, scope `create_receipt`). Posée par la migration **064** (`employee` seul, sous le nom `profiles_employee_requires_establishment`) puis étendue et renommée par la **065** (`+ establishment`). Validée immédiatement dans les deux cas : 0 violation en base. Une violation lève `23514`. S'applique à **toutes** les écritures, `service_role` inclus. Seule `handle_new_user` écrit `attached_establishment_id` en base, avec `role` au défaut `client` : les inscriptions ne sont pas concernées. |
