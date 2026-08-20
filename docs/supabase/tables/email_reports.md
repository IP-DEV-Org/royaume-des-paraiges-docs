# Tables : rapports e-mail automatisés

Trio de tables introduit par la **migration 076** (août 2026) : configuration, destinataires et journal des envois de rapports chiffrés récurrents. Piloté depuis la page `/reports` du dashboard admin, exécuté par l'Edge Function [`send-email-reports`](../edge-functions/README.md) et le cron `email-reports-dispatch` (migration 078).

> ⚠️ **Reporting interne uniquement.** Les destinataires sont des adresses e-mail libres (équipe, gérants), **sans lien avec `auth.users`**. À ce titre, aucun consentement ni lien de désinscription n'est nécessaire. Y mettre des adresses de **clients de l'app** changerait la nature du traitement : cela imposerait un opt-in, un lien de désinscription et une mise à jour de la Politique de confidentialité et des CGU. Ne pas détourner ces tables pour cela.

## Vue d'ensemble

```
email_reports (1) ──< email_report_recipients (N)
      │
      └──< email_report_runs (N)
```

Trois rapports sont livrés en seed, tous **inactifs** par défaut :

| `key` | `report_type` | `period_type` | Contenu |
|---|---|---|---|
| `monthly_activity` | `activity_summary` | `monthly` | CA, tickets, panier moyen, répartition par établissement, inscrits, joueurs actifs, montées de niveau, répartition par rang |
| `weekly_leaderboard` | `leaderboard` | `weekly` | Podium, top N, participants, XP distribué |
| `monthly_leaderboard` | `leaderboard` | `monthly` | Idem, sur le mois |

Ajouter un rapport = une ligne ici + un builder SQL dans [`get_email_report_payload`](../functions/get_email_report_payload.md) + un gabarit de rendu dans l'Edge Function.

## `email_reports`

| Colonne | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | `uuid` | Non | `gen_random_uuid()` | PK |
| `key` | `text` | Non | - | Identifiant stable, UNIQUE, CHECK `^[a-z0-9_]+$`. Consommé par l'Edge Function et l'URL `/reports/[key]`. Ne pas renommer une clé déjà référencée. |
| `report_type` | `text` | Non | - | CHECK `activity_summary` / `leaderboard`. Détermine le builder SQL et le gabarit HTML. |
| `period_type` | `text` | Non | - | CHECK `weekly` / `monthly`. Le rapport porte **toujours sur la période écoulée**, jamais sur la période en cours. |
| `name` | `text` | Non | - | Titre affiché dans l'admin et en tête de l'e-mail. |
| `description` | `text` | Oui | - | Résumé affiché sur la carte de la page `/reports`. |
| `subject_template` | `text` | Non | - | Objet de l'e-mail. `{{period_label}}` est remplacé par le libellé humain de la période. |
| `options` | `jsonb` | Non | `{}` | Réglages par type. `leaderboard` : `{"top_n": 10}`. |
| `is_active` | `boolean` | Non | `false` | `false` = ignoré par le cron. Le bouton « Envoyer maintenant » fonctionne quel que soit ce drapeau. |
| `last_period_sent` | `text` | Oui | - | Dernière période envoyée (`2026-W29`, `2026-07`). **Porte l'idempotence** : le cron saute un rapport dont la période cible vaut déjà cette valeur. |
| `last_run_at` | `timestamptz` | Oui | - | Horodatage du dernier envoi réussi. |
| `created_at` / `updated_at` | `timestamptz` | Non | `now()` | `updated_at` via trigger `set_updated_at`. |

### Triggers

- **`trg_email_reports_updated_at`** (BEFORE UPDATE) : `set_updated_at()`.
- **`trg_email_report_activation`** (BEFORE UPDATE OF `is_active`) : à la transition `false → true`, positionne `last_period_sent` sur la période écoulée via `get_previous_period_identifier()`. **Activer un rapport n'envoie donc jamais rétroactivement** la période déjà close ; pour l'envoyer quand même, utiliser « Envoyer maintenant ».

## `email_report_recipients`

| Colonne | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | `uuid` | Non | `gen_random_uuid()` | PK |
| `report_id` | `uuid` | Non | - | FK → `email_reports.id` (ON DELETE CASCADE) |
| `email` | `text` | Non | - | CHECK de forme permissif (pas de validation TLD). Normalisée en minuscules par trigger. |
| `label` | `text` | Oui | - | Libellé libre (« Gérant La Chapelle »). |
| `is_active` | `boolean` | Non | `true` | `false` = conservé dans la liste mais exclu de l'envoi (pause sans perte du libellé). |
| `created_at` | `timestamptz` | Non | `now()` | Ordre d'affichage et d'envoi. |

- **UNIQUE (`report_id`, `email`)** : réellement insensible à la casse grâce au trigger de normalisation.
- **`trg_email_report_recipient_normalize`** (BEFORE INSERT/UPDATE OF `email`) : `lower(btrim(email))`.
- Index partiel `idx_email_report_recipients_report` sur (`report_id`) WHERE `is_active`.

## `email_report_runs`

Journal des tentatives, une ligne par (rapport, exécution). **Écrit exclusivement par l'Edge Function en `service_role`** : aucune policy d'écriture `authenticated` n'existe.

| Colonne | Type | Nullable | Description |
|---|---|---|---|
| `id` | `bigint` | Non | PK identity |
| `report_id` | `uuid` | Non | FK → `email_reports.id` (ON DELETE CASCADE) |
| `period_identifier` | `text` | Non | Période couverte par l'envoi |
| `status` | `text` | Non | CHECK `success` / `partial` / `error` / `skipped` |
| `trigger_source` | `text` | Non | CHECK `cron` / `manual` / `test` |
| `sent_count` / `failed_count` | `integer` | Non | Compteurs par destinataire |
| `error_message` | `text` | Oui | Message du provider, agrégé par adresse en cas d'échec partiel |
| `payload` | `jsonb` | Oui | Snapshot des données envoyées : permet d'auditer un chiffre contesté a posteriori sans recalculer |
| `triggered_by` | `uuid` | Oui | FK → `profiles.id` (ON DELETE SET NULL). NULL pour les passages du cron. |
| `started_at` / `finished_at` | `timestamptz` | Non / Oui | - |

Sémantique des statuts :

- `success` : tous les destinataires servis ;
- `partial` : au moins un échec, au moins un succès ;
- `error` : rien n'est parti (calcul des données impossible, secrets Resend absents, provider KO) ;
- `skipped` : période déjà envoyée, ou aucun destinataire actif.

Index `idx_email_report_runs_report_time` sur (`report_id`, `started_at DESC`).

## RLS

Les trois tables ont RLS activée, **admin-only**.

| Table | SELECT | INSERT / UPDATE / DELETE |
|---|---|---|
| `email_reports` | `profiles.role = 'admin'` | `admin_has_feature('reports')` |
| `email_report_recipients` | `profiles.role = 'admin'` | `admin_has_feature('reports')` |
| `email_report_runs` | `profiles.role = 'admin'` | *(aucune policy : service_role uniquement)* |

La lecture reste ouverte à tout admin, même privé de la fonctionnalité : le masquage de la page est assuré par le middleware. L'écriture, elle, applique le **feature gating en RLS** (même modèle que la migration 070 sur les quêtes) : sans quoi un admin restreint pourrait reconfigurer les envois via un appel REST direct.

## Consommateurs

- **Dashboard admin `/reports`** : liste, activation, CRUD destinataires, envoi manuel, envoi de test, prévisualisation, historique. Service `src/lib/services/emailReportService.ts`.
- **Edge Function `send-email-reports`** : lit la config, écrit les runs, met à jour `last_period_sent`.
- **Cron `email-reports-dispatch`** (migration 078, 07:00 UTC quotidien) : déclenche l'Edge Function sans corps de requête.
