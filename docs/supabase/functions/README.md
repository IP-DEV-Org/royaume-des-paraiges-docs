# Fonctions PostgreSQL

Cette section documente toutes les fonctions PostgreSQL disponibles dans le schema `public`.

## Liste des fonctions (62)

> **Note** : les 14 fonctions du système **achievement badges** (migrations 022-026) sont documentées séparément dans [`achievement_badges.md`](./achievement_badges.md). Il s'agit de 6 `check_achievement_*` + 6 `progress_achievement_*` + 2 dispatchers (`award_achievements_for_user`, `award_achievements_for_all_for_badge`) + 1 cron (`award_achievements_for_all_cron`) + 3 RPCs front (`get_achievement_progress`, `get_unseen_badges`, `mark_badges_seen`).
>
> Pour les triggers de redondance des quêtes (migration 021), voir [`check_quest_redundancy.md`](./check_quest_redundancy.md).
>
> **Migration 040 (Security Definer hardening, 18/05/2026)** : 3 helpers `assert_*` ajoutés (`assert_admin`, `assert_admin_or_establishment_for`, `assert_self_or_staff`), nouvelle RPC publique `get_period_preview_public` pour le front (zéro UUID exposé), `p_admin_id` retiré des signatures de `create_manual_coupon`, `distribute_quest_reward`, `distribute_period_rewards_v2` (audit trail dérivé de `auth.uid()`). `credit_bonus_cashback` et `check_email_exists` sont passés en **REVOKE total** côté REST (uniquement chain interne / `service_role`). Voir [`assert_admin.md`](./assert_admin.md) et la migration `supabase/migrations/040_security_definer_hardening.sql`.

| Fonction | Arguments | Retour | Volatilite | Security Definer |
|----------|-----------|--------|------------|------------------|
| `assert_admin` | - | `void` | VOLATILE | Oui |
| `assert_admin_or_establishment_for` | p_establishment_id bigint | `void` | VOLATILE | Oui |
| `assert_self_or_staff` | p_customer_id uuid | `void` | VOLATILE | Oui |
| `award_user_badge` | p_customer_id uuid, p_badge_slug character varying, p_period_type character varying, p_period_identifier character varying, p_rank bigint | `json` | VOLATILE | Oui |
| `calculate_gains` | p_amount_for_gains integer | `jsonb` | VOLATILE | Non |
| `calculate_quest_progress` | p_customer_id uuid, p_quest_id bigint, p_period_identifier character varying DEFAULT NULL::character varying | `integer` | VOLATILE | Oui |
| `check_cashback_balance` | p_customer_id uuid, p_cashback_requested integer | `jsonb` | VOLATILE | Non |
| `check_email_exists` | email_to_check text | `boolean` | VOLATILE | Oui |
| `check_period_closed` | p_period_type character varying, p_period_identifier character varying | `boolean` | VOLATILE | Non |
| `credit_bonus_cashback` | p_customer_id uuid, p_amount integer, p_coupon_id bigint DEFAULT NULL, p_source_type varchar DEFAULT 'bonus_cashback_manual', p_period_identifier varchar DEFAULT NULL | `bigint` | VOLATILE | Oui |
| `create_frequency_coupon` | p_customer_id uuid | `json` | VOLATILE | Oui |
| `create_leaderboard_reward_coupon` | p_customer_id uuid, p_amount integer DEFAULT NULL::integer, p_percentage integer DEFAULT NULL::integer | `json` | VOLATILE | Oui |
| `create_manual_coupon` | p_customer_id uuid, p_template_id bigint DEFAULT NULL::bigint, p_amount integer DEFAULT NULL::integer, p_percentage integer DEFAULT NULL::integer, p_expires_at timestamp with time zone DEFAULT NULL::timestamp with time zone, p_validity_days integer DEFAULT NULL::integer, p_notes text DEFAULT NULL::text | `jsonb` | VOLATILE | Oui |
| `create_receipt` | p_customer_id uuid, p_establishment_id bigint, p_payment_methods jsonb, p_coupon_ids bigint[] DEFAULT ARRAY[]::bigint[] | `jsonb` | VOLATILE | Oui |
| `create_spending_from_cashback_payment` | - | `trigger` | VOLATILE | Non |
| `create_weekly_coupon` | p_customer_id uuid | `json` | VOLATILE | Oui |
| `distribute_all_quest_rewards` | p_admin_id uuid DEFAULT NULL::uuid | `json` | VOLATILE | Oui |
| `distribute_leaderboard_rewards` | p_period_type character varying, p_force boolean DEFAULT false | `json` | VOLATILE | Oui |
| `distribute_period_rewards_v2` | p_period_type character varying, p_period_identifier character varying DEFAULT NULL::character varying, p_preview_only boolean DEFAULT false, p_force boolean DEFAULT false | `jsonb` | VOLATILE | Oui |
| `distribute_quest_reward` | p_quest_progress_id bigint | `json` | VOLATILE | Oui |
| `distribute_quest_rewards` | - | `trigger` | VOLATILE | Non |
| `get_coupon_stats` | - | `jsonb` | VOLATILE | Oui |
| `get_current_user_role` | - | `text` | VOLATILE | Oui |
| `get_customer_available_coupons` | p_customer_id uuid | `TABLE(id bigint, created_at timestamp with time zone, customer_id uuid, used boolean, amount integer, percentage integer)` | VOLATILE | Oui |
| `get_period_bounds` | p_period_type character varying, p_period_identifier character varying | `TABLE(period_start timestamp with time zone, period_end timestamp with time zone)` | IMMUTABLE | Non |
| `get_period_identifier` | p_period_type character varying, p_date timestamp with time zone DEFAULT now() | `character varying` | IMMUTABLE | Non |
| `get_period_preview` | p_period_type character varying, p_period_identifier character varying DEFAULT NULL::character varying | `jsonb` | VOLATILE | Oui |
| `get_period_preview_public` | p_period_type character varying, p_period_identifier character varying, p_top_n integer DEFAULT 20 | `TABLE(rank bigint, username text, avatar_url text, current_xp bigint, projected_reward_amount integer, projected_reward_type text, is_me boolean)` | VOLATILE | Oui |
| `get_user_badges` | p_customer_id uuid | `TABLE(badge_id integer, slug character varying, name character varying, description text, icon character varying, rarity character varying, category character varying, earned_at timestamp with time zone, period_type character varying, period_identifier character varying, rank integer)` | VOLATILE | Non |
| `get_user_cashback_balance` | p_customer_id uuid | `jsonb` | STABLE | Non |
| `get_user_complete_stats` | p_customer_id uuid | `jsonb` | STABLE | Non |
| `get_user_info` | user_ids uuid[] | `TABLE(id uuid, email text, first_name text, last_name text, username text, avatar_url text)` | VOLATILE | Oui |
| `get_user_quests` | p_customer_id uuid, p_period_type character varying DEFAULT NULL::character varying | `json` | VOLATILE | Oui |
| `get_user_xp_stats` | p_customer_id uuid | `jsonb` | STABLE | Non |
| `handle_new_user` | - | `trigger` | VOLATILE | Oui |
| `set_updated_at` | - | `trigger` | VOLATILE | Non |
| `handle_user_delete` | - | `trigger` | VOLATILE | Oui |
| `sync_auth_to_profiles` | - | `TABLE(synced_count integer, user_ids text[])` | VOLATILE | Oui |
| `trigger_update_quest_progress` | - | `trigger` | VOLATILE | Oui |
| `update_meta_quest_progress` | p_customer_id uuid, p_completed_quest_period_type varchar | `void` | VOLATILE | Oui |
| `update_profile_from_auth` | user_id uuid | `void` | VOLATILE | Oui |
| `update_quest_progress_for_receipt` | p_receipt_id bigint | `json` | VOLATILE | Oui |
| `validate_coupons` | p_customer_id uuid, p_coupon_ids bigint[] | `jsonb` | VOLATILE | Non |
| `validate_payment_methods` | p_payment_methods jsonb | `jsonb` | VOLATILE | Non |

## Descriptions


### award_user_badge

Attribue un badge à un utilisateur pour une période donnée

- **Arguments**: `p_customer_id uuid, p_badge_slug character varying, p_period_type character varying, p_period_identifier character varying, p_rank bigint`
- **Retour**: `json`


### calculate_quest_progress

Calcule la progression actuelle d'un utilisateur pour une quete donnee. Supporte les types `xp_earned`, `amount_spent` (déprécié), `cashback_earned`, `establishments_visited`, `orders_count`, `quest_completed` et `consumption_count`. Pour `cashback_earned` (migration 029), progression = SUM(`gains.cashback_money`) sur la période — coefficient client et bonus coupons inclus. Pour `quest_completed`, compte le nombre de sous-périodes distinctes (weekly pour monthly, monthly pour yearly) avec au moins 1 quête complétée dans `quest_completion_logs`.

- **Arguments**: `p_customer_id uuid, p_quest_id bigint, p_period_identifier character varying DEFAULT NULL::character varying`
- **Retour**: `integer`


### check_period_closed

Vérifie si une période de leaderboard a déjà été fermée

- **Arguments**: `p_period_type character varying, p_period_identifier character varying`
- **Retour**: `boolean`


### credit_bonus_cashback

Credite un bonus cashback directement au solde d'un utilisateur via la table `gains`. Voir [credit_bonus_cashback.md](./credit_bonus_cashback.md) pour la documentation complete.

Depuis la migration 040, **REVOKE EXECUTE FROM PUBLIC, anon, authenticated** : la fonction n'est plus appelable via REST. Reste accessible uniquement via la chaîne SECURITY DEFINER interne (depuis `create_manual_coupon`, `distribute_*`, trigger `distribute_quest_rewards`) ou via `service_role`.

- **Arguments**: `p_customer_id uuid, p_amount integer, p_coupon_id bigint DEFAULT NULL, p_source_type varchar DEFAULT 'bonus_cashback_manual', p_period_identifier varchar DEFAULT NULL`
- **Retour**: `bigint` (id du gain cree)


### create_frequency_coupon

Crée automatiquement un coupon de 5% de réduction pour un utilisateur qui a passé au moins 10 commandes durant la semaine actuelle (lundi-dimanche). Un seul coupon par semaine est autorisé.

- **Arguments**: `p_customer_id uuid`
- **Retour**: `json`


### create_leaderboard_reward_coupon

Crée un coupon de récompense leaderboard (montant OU pourcentage)

- **Arguments**: `p_customer_id uuid, p_amount integer DEFAULT NULL::integer, p_percentage integer DEFAULT NULL::integer`
- **Retour**: `json`


### create_receipt

Crée un reçu avec paiements, coupons et gains. Applique les coefficients XP et cashback du profil client (100 = 1x, 150 = 1.5x, 50 = 0.5x).

- **Arguments**: `p_customer_id uuid, p_establishment_id bigint, p_payment_methods jsonb, p_coupon_ids bigint[] DEFAULT ARRAY[]::bigint[]`
- **Retour**: `jsonb`


### create_spending_from_cashback_payment

Fonction déclenchée automatiquement lors de l'insertion d'une receipt_line. 
Si payment_method = 'cashback', crée automatiquement une entrée dans spendings.

- **Arguments**: `aucun`
- **Retour**: `trigger`


### create_weekly_coupon

Crée automatiquement un coupon de 3,90€ pour un utilisateur qui a dépensé plus de 50€ durant la semaine actuelle (lundi-dimanche). Un seul coupon par semaine est autorisé.

- **Arguments**: `p_customer_id uuid`
- **Retour**: `json`


### distribute_all_quest_rewards

Distribue les recompenses pour toutes les quetes completees en attente

- **Arguments**: `p_admin_id uuid DEFAULT NULL::uuid`
- **Retour**: `json`


### distribute_leaderboard_rewards

Distribue automatiquement les récompenses aux TOP 10 du leaderboard

- **Arguments**: `p_period_type character varying, p_force boolean DEFAULT false`
- **Retour**: `json`


### distribute_period_rewards_v2

Distribue les récompenses leaderboard avec support des tiers configurables et mode preview. Admin only depuis la migration 040 (`assert_admin()` + REVOKE anon).

- **Arguments**: `p_period_type character varying, p_period_identifier character varying DEFAULT NULL::character varying, p_preview_only boolean DEFAULT false, p_force boolean DEFAULT false`
- **Retour**: `jsonb`


### distribute_quest_reward

Distribue les recompenses pour une quete completee. Admin only depuis la migration 040 (`assert_admin()` + REVOKE anon).

- **Arguments**: `p_quest_progress_id bigint`
- **Retour**: `json`


### distribute_quest_rewards

Fonction trigger declenchee quand le statut d'un `quest_progress` passe a `completed`. Cree automatiquement les coupons (bonus cashback si montant fixe), attribue les badges, donne les bonus XP/cashback directs, et logue dans `quest_completion_logs`. Met le statut a `rewarded` et `rewarded_at` apres traitement. Appelle ensuite `update_meta_quest_progress()` pour propager la completion aux meta-quetes (`quest_completed`).

- **Arguments**: `aucun`
- **Retour**: `trigger`


### get_coupon_stats

Retourne les statistiques globales des coupons pour le dashboard admin

- **Arguments**: `aucun`
- **Retour**: `jsonb`


### get_customer_available_coupons

Récupère les coupons disponibles (non utilisés) d'un client. 
Accessible uniquement aux employees, establishments et admins.
Utilisée lors du scan QR code client pour le paiement.

- **Arguments**: `p_customer_id uuid`
- **Retour**: `TABLE(id bigint, created_at timestamp with time zone, customer_id uuid, used boolean, amount integer, percentage integer)`


### get_period_identifier

Calcule l'identifiant de période (2026-W04, 2026-01, 2026) pour un type et une date donnés

- **Arguments**: `p_period_type character varying, p_date timestamp with time zone DEFAULT now()`
- **Retour**: `character varying`


### get_period_preview

Prévisualise la distribution des récompenses sans les créer

- **Arguments**: `p_period_type character varying, p_period_identifier character varying DEFAULT NULL::character varying`
- **Retour**: `jsonb`


### get_user_badges

Récupère tous les badges d'un utilisateur avec leurs détails complets

- **Arguments**: `p_customer_id uuid`
- **Retour**: `TABLE(badge_id integer, slug character varying, name character varying, description text, icon character varying, rarity character varying, category character varying, earned_at timestamp with time zone, period_type character varying, period_identifier character varying, rank integer)`


### get_user_cashback_balance

Retourne le solde cashback d'un utilisateur depuis la vue matérialisée.
Utilisation: SELECT get_user_cashback_balance('11111111-1111-1111-1111-111111111111'::UUID);

- **Arguments**: `p_customer_id uuid`
- **Retour**: `jsonb`


### get_user_complete_stats

Retourne toutes les statistiques d'un utilisateur depuis la vue matérialisée complète.
Utilisation: SELECT get_user_complete_stats('11111111-1111-1111-1111-111111111111'::UUID);

- **Arguments**: `p_customer_id uuid`
- **Retour**: `jsonb`


### get_user_quests

Recupere toutes les quetes actives avec la progression de l'utilisateur pour la periode courante

- **Arguments**: `p_customer_id uuid, p_period_type character varying DEFAULT NULL::character varying`
- **Retour**: `json`


### get_user_xp_stats

Retourne les XP et statistiques d'un utilisateur depuis la vue matérialisée.
Utilisation: SELECT get_user_xp_stats('11111111-1111-1111-1111-111111111111'::UUID);

- **Arguments**: `p_customer_id uuid`
- **Retour**: `jsonb`


### handle_new_user

Fonction trigger qui crée automatiquement un profil dans public.profiles lors de la création d'un utilisateur dans auth.users

- **Arguments**: `aucun`
- **Retour**: `trigger`


### set_updated_at

Fonction trigger generique qui met a jour la colonne `updated_at` a `now()` lors de la modification d'une ligne. Utilisee sur les tables de contenu de contenu (breweries, establishments, beer_styles, beers, news, level_thresholds, etc.).

- **Arguments**: `aucun`
- **Retour**: `trigger`


### handle_user_delete

Supprime automatiquement le profil dans public.profiles quand un utilisateur est supprimé de auth.users

- **Arguments**: `aucun`
- **Retour**: `trigger`


### sync_auth_to_profiles

Synchronise tous les utilisateurs de auth.users vers public.profiles. Retourne le nombre d'utilisateurs synchronisés et leurs IDs.

- **Arguments**: `aucun`
- **Retour**: `TABLE(synced_count integer, user_ids text[])`


### update_profile_from_auth

Met à jour un profil existant avec les données de auth.users

- **Arguments**: `user_id uuid`
- **Retour**: `void`


### update_meta_quest_progress

Met à jour la progression des méta-quêtes (`quest_completed`) suite à la complétion d'une quête. Détermine le type de période parent (weekly→monthly, monthly→yearly), puis pour chaque quête active de type `quest_completed` du bon period_type, recalcule la progression et crée/met à jour l'entrée `quest_progress`. La chaîne est naturellement limitée : weekly → monthly → yearly → fin.

- **Arguments**: `p_customer_id uuid, p_completed_quest_period_type varchar`
- **Retour**: `void`


### update_quest_progress_for_receipt

Met a jour la progression de toutes les quetes apres un nouveau receipt

- **Arguments**: `p_receipt_id bigint`
- **Retour**: `json`

