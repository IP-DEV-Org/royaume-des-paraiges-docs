# Documentation Supabase

## Introduction

Cette section documente l'utilisation de Supabase comme backend pour le Royaume des Paraiges.

**Project ID**: `kioysoveqemzjolfwpnu` (IPDEV — migration le 13/05/2026, en phase d'observation)
**URL API**: `https://kioysoveqemzjolfwpnu.supabase.co`
**Region**: eu-west-3 (Paris)
**PostgreSQL**: 17.6.1.003

> **Note historique** : le projet Royaume `uflgfsoekkgegdgecubb` (us-east-2) reste référencé dans les anciens dumps et scripts. Toute nouvelle requête doit cibler IPDEV.

## Table des matieres

- [Configuration complete](./configuration.md) - Schema, tables, fonctions, triggers

## Resume

- **42 tables** public avec RLS active (table `notes` supprimée 18/05/2026, table `user_legal_consents` restaurée 18/05/2026)
- **97 fonctions** PostgreSQL — incluant RPCs sécurisées (mai 2026) remplaçant les accès directs aux MV, fonctions RGPD (mai 2026), gestion photo d'identité avec cooldown, et garde-fou solde PdB non négatif (`enforce_non_negative_cashback`, migration 043)
- **4 vues materialisees** (`weekly_xp_leaderboard`, `monthly_xp_leaderboard`, `yearly_xp_leaderboard`, `user_stats`) — **fermées à l'API PostgREST** depuis 18/05/2026 ; lisibles uniquement via les RPC wrappers.
- **6 vues** SQL (cashpad_health_*, public_profiles, avg_ticket_12m, reward_distribution_stats) — passées en `security_invoker=true` 18/05/2026.
- **20 triggers** (8 métier + 2 validation + 10 auto-timestamp) — voir `triggers/README.md`
- **4 jobs pg_cron** pour distributions automatiques (dont `award_achievements_cron` quotidien 02:00 UTC)
- **2 buckets storage** (avatars, content-assets) — policies durcies 18/05/2026 (plus de listing public, plus d'écrasement d'avatars d'autrui)
- **3 edge functions** (cashpad-webhook, cashpad-process-queue, cashpad-reconcile-daily) — voir `edge-functions/README.md`
- **4 enums** personnalises (consumption_type, payment_method, quest_type, user_role)

## Tables par Categorie

### Utilisateurs & Auth
- `profiles` - Profils utilisateurs
- `user_badges` - Badges obtenus
- `gdpr_requests` - Demandes RGPD (export / suppression)

### Coupons & Recompenses
- `coupons` - Coupons utilisateurs
- `coupon_templates` - Modèles de coupons
- `coupon_distribution_logs` - Historique distributions
- `reward_tiers` - Paliers de récompenses

### Quetes
- `quests` - Définition des quêtes
- `quest_progress` - Progression utilisateurs
- `quest_completion_logs` - Historique complétions
- `quest_periods` - Liaison quêtes-périodes
- `available_periods` - Périodes disponibles

### Leaderboard
- `leaderboard_reward_distributions` - Historique récompenses
- `period_closures` - Périodes fermées
- `period_reward_configs` - Config personnalisée
- `season_closure_log` - Journal des étapes de clôture de saison
- `season_snapshots` - Photographie immuable rang/XP/coefficient par saison

### Transactions
- `receipts` - Tickets de caisse
- `receipt_lines` - Lignes de paiement
- `receipt_consumption_items` - Types de consommation (optionnel)
- `gains` - Gains XP/cashback
- `spendings` - Dépenses cashback

### Contenu
- `beers` - Catalogue bières
- `breweries` - Brasseries
- `beer_styles` - Styles de bières
- `establishments` - Établissements
- `establishment_groups` - Groupes d'établissements (caisse Cashpad commune)
- `establishment_consumption_types` - Types de consommation par établissement
- `news` - Actualités
- `level_thresholds` - Seuils de niveaux

### Social
- `likes` - Likes
- `comments` - Commentaires
- ~~`notes`~~ — **Supprimée 18/05/2026** (orpheline, 0 ligne, jamais utilisée)

### Autres
- `badge_types` - Types de badges (9 classement + 6 saison + 5+ succès)
- `constants` - Constantes système

### Réconciliation Cashpad
- `cashpad_receipts_snapshot` - Cache des tickets Cashpad (archives API)
- `cashpad_reconciliations` - Liens de réconciliation receipt Royaume ↔ ticket Cashpad
- `cashpad_matching_params` - Paramètres adaptatifs de matching par établissement
- `cashpad_employee_mappings` - Mapping profils Royaume ↔ serveurs Cashpad
- `cashpad_webhook_queue` - Queue des événements webhook Cashpad

### Configuration
- `legal_pages` - Pages légales (colonne `version` — bump force re-acceptation)
- `user_legal_consents` - Journal d'acceptation versionnée des documents légaux (immuable, RLS user-self)
- `admin_settings` - Paramétrage admin key-value JSONB

### Tables de liaison (M2M)
- `beers_establishments` - Bières-Établissements
- `beers_beer_styles` - Bières-Styles
- `news_establishments` - News-Établissements
- `quests_establishments` - Scoping quêtes par établissement (aucune entrée = quête globale)

## Migrations

Les migrations SQL sont appliquées automatiquement. Voir `supabase_migrations.schema_migrations` pour la liste complète. Dernières migrations notables :
- **Mai 2026** : identity_photo_feature, GDPR (anonymisation + rétention), establishment_consumption_types, identity_photo_cooldown, level_thresholds_admin_rls
- **18 mai 2026** : Hardening sécurité (16 migrations), establishment_groups
- **Mai 2026** : Réconciliation Cashpad (migrations 032-036)
- **Avril 2026** : Badges succès (022-027, 030-031), cashback_earned quest type (028-029), prévention quêtes redondantes (020-021)

## Ressources

- [Documentation officielle Supabase](https://supabase.com/docs)
- [API Reference](https://supabase.com/docs/reference)
- [Dashboard](https://app.supabase.com)

## Derniere mise a jour

### Hardening sécurité post-migration IPDEV (16-18 mai 2026)

Audit Supabase advisors complet : **180 warnings → 38 warnings, 0 ERROR**. Tous les warnings restants sont intentionnels et documentés (RPCs publiques légitimes + check rôle interne dans les fonctions admin/self-only que le linter ne sait pas inspecter).

**Migrations BDD appliquées** (16 migrations, sans préfixe numérique — voir `supabase_migrations.schema_migrations`) :

- `restore_constants_public_read_policy` — fix bug `/storytelling` (policy SELECT sur `constants` perdue au pg_dump)
- `security_drop_orphan_notes_table` — DROP table `notes` (orpheline, 0 ligne)
- `security_add_missing_admin_policies` — policies admin SELECT sur `cashpad_webhook_queue`, `season_closure_log`, `season_snapshots`
- `security_invoker_views_and_revoke_anon` — 5 vues passées en `security_invoker=true`, REVOKE anon sur vues admin
- `security_set_function_search_path` — `SET search_path = public, pg_temp` sur 36 fonctions (anti privilege-escalation via shadowing `pg_temp`)
- `security_revoke_internal_function_execute_public` — REVOKE EXECUTE FROM PUBLIC sur 34 fonctions internes
- `security_restrict_authenticated_only_functions` + `security_revoke_anon_from_admin_functions` — REVOKE FROM anon sur 26 RPCs admin/authenticated
- `restore_badge_types_admin_policy` — policy admin perdue au pg_dump (fix 403 création de badge)
- `resolve_credit_bonus_cashback_overload_ambiguity` — DROP `credit_bonus_cashback(uuid,integer,bigint,varchar)` 4-arg (fix erreur 400 ambiguïté)
- `security_storage_drop_laxist_avatar_policies` — fix trou de sécu : n'importe quel user pouvait écraser l'avatar d'autrui
- `security_storage_block_bucket_listing` — DROP policies publiques de LIST sur buckets `avatars` et `content-assets`
- `security_user_stats_rpc_wrappers` + `security_revoke_user_stats_matview` + `security_revoke_anon_from_new_rpc_wrappers` — création de `get_public_xp_leaderboard`, `get_public_user_xp`, `get_unspent_cashback_total` + fermeture de la MV `user_stats`
- `security_revoke_xp_leaderboard_matviews` (18/05) — création de [`get_current_xp_leaderboard`](./functions/get_current_xp_leaderboard.md) et [`get_current_xp_rank`](./functions/get_current_xp_rank.md), fermeture des 3 MV `*_xp_leaderboard` (filtrage `*_total_spent` au passage : règle « zéro euro côté client »)
- `restore_legal_versioning_structure` (18/05) — restauration de `legal_pages.version` et de la table [`user_legal_consents`](./tables/user_legal_consents.md), perdues au pg_dump du 13/05. Backfill v1.0 depuis `profiles.terms_accepted_at` (filtré par jointure `auth.users` pour ignorer les 8 profils anonymisés RGPD).

**Vues SECURITY DEFINER corrigées** (5 vues passées en `security_invoker=true`) :
- `public_profiles`, `avg_ticket_12m`, `cashpad_clock_drift`, `cashpad_window_feedback`, `cashpad_health_stats_30d`

**Activé manuellement côté Dashboard** :
- HaveIBeenPwned password protection (Authentication → Sign In / Up → Email → Password Protection)

**Code applicatif refactoré** :
- `royaume-paraiges-front/src/features/gains/services/leaderboardService.ts` : 6 `from('*_xp_leaderboard')` → 2 helpers RPC privés (`fetchCurrentLeaderboard`, `fetchUserRank`)
- `royaume-paraiges-admin/src/lib/services/userService.ts` : 3 `from('*_xp_leaderboard')` → 3 `rpc('get_current_xp_rank')`
- `royaume-paraiges-front/src/features/auth/services/clientService.ts` : `from('user_stats')` → `rpc('get_public_user_xp')`
- `royaume-paraiges-scanner/src/lib/services/clientService.ts` : `from('user_stats').single()` → `rpc('get_user_stats')` avec mapping array→object
- `royaume-paraiges-admin/src/lib/services/analyticsService.ts` : `from('user_stats')` + agg client-side → `rpc('get_unspent_cashback_total')`
- `royaume-paraiges-admin/src/app/(dashboard)/quests/_form/QuestForm.tsx` : Zod `superRefine` pour valider qu'au moins une récompense est configurée (fix 400 BDD `quests.has_reward`)
- Scripts `supabase:types` corrigés (front + scanner pointaient encore vers l'ancien `uflgfsoekkgegdgecubb`)

### Migrations mai 2026 (post-hardening)

- **`identity_photo_feature`** (26/05) — Colonnes `identity_photo_url` et `identity_photo_updated_at` sur `profiles`, bucket storage pour photos d'identité
- **`gdpr_add_identity_photo_cleanup`** (26/05) — Nettoyage photo d'identité dans `gdpr_anonymize_user`
- **`legal_bump_v1_2_identity_photo`** (26/05) — Bump version CGU v1.2 suite ajout photo d'identité
- **`level_thresholds_admin_rls`** (26/05) — Policies admin CRUD sur `level_thresholds`
- **`create_establishment_consumption_types`** (26/05) — Table `establishment_consumption_types` avec policies RLS
- **`identity_photo_cooldown`** (27/05) — Trigger `enforce_identity_photo_cooldown` (30 jours) + RPC `admin_reset_identity_photo_cooldown`
- **`043_guard_negative_cashback_balance`** (31/05) — Trigger `trg_enforce_non_negative_cashback` sur `gains` + fonction `enforce_non_negative_cashback` : empêche tout solde de Paraiges de Bronze négatif (errcode `P0423`). Voir `functions/enforce_non_negative_cashback.md`

### Historique antérieur

- **Date**: 2026-04-22
- **Migration 020** — Prerequis prevention quetes redondantes : tables `quests_establishments`, `admin_settings`, vue `avg_ticket_12m`.
- **Migration 021** — Triggers de detection : fonction `check_quest_redundancy` + 3 triggers (errcode `P0421`) bloquant toute activation/scope creant une redondance.
- **Migration 022** — Schéma « badges succès » : extension CHECK sur `badge_types.category` et `user_badges.period_type` (ajout `achievement`), colonnes `criterion_type`/`criterion_params`/`evaluation_mode`/`archived_at` sur `badge_types`, `seen_at` sur `user_badges`, index unique partiel achievement, back-fill seen_at.
- **Migration 023** — 12 fonctions de check et de progression (une par `criterion_type`).
- **Migration 024** — Dispatchers + RPCs achievement (`award_achievements_for_user`, `award_achievements_for_all_for_badge`, `award_achievements_for_all_cron`, `get_achievement_progress`, `get_unseen_badges`, `mark_badges_seen`).
- **Migration 025** — Seed 5 badges achievement + attribution rétroactive + pré-marquage `seen_at`.
- **Migration 026** — `create_receipt` patchée avec appel `award_achievements_for_user` (étape 12b) protégé par EXCEPTION local.
- **Migration 027** — pg_cron `award_achievements_cron` planifié à `0 2 * * *`.
- **Migration 030** (2026-04-23) — Durcissement RLS des 5 RPCs achievement : check `auth.uid() = p_customer_id` (ou admin/employee/establishment selon la RPC) ; REVOKE EXECUTE du cron pour PUBLIC/authenticated/anon.
- **Migration 031** (2026-04-23) — FK `user_badges.badge_id` passée de `ON DELETE CASCADE` à `ON DELETE RESTRICT` pour forcer l'usage du soft-delete (`badge_types.archived_at`).
- Détails complets : cf. `functions/achievement_badges.md`.
