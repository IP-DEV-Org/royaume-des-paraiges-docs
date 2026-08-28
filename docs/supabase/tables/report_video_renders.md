# Table: report_video_renders

## Description

État du rendu de la **vidéo de classement** attachée aux rapports e-mail. Les rapports `weekly_leaderboard` et `monthly_leaderboard` peuvent embarquer une vidéo générée depuis un template **HyperFrames**, jointe à l'e-mail et réutilisable sur les réseaux sociaux.

Le rendu lui-même ne tourne pas dans Supabase : il a lieu dans un **Vercel Sandbox** piloté par le projet `royaume-video-renderer`. Cette table est le **point de rendez-vous** entre les deux moitiés du pipeline.

Introduite par la migration **082 (28/08/2026)**.

## Ce que la table ne contient pas

Volontairement absents, parce que dérivables :

- **Les données du classement** : recalculées à la volée par `get_report_video_variables`. Rien n'est dupliqué.
- **L'URL du MP4** : déterministe, `report-videos/<report_key>/<period_identifier>.mp4`.
- **Le poids et la durée du fichier** : déjà dans les métadonnées Storage.

Ne subsiste que l'état du rendu, qui ne se recalcule pas.

## Grain

Une ligne par `(report_id, period_identifier)`, **pas par tentative** : le compteur `attempts` porte les reprises. Soit ~52 lignes/an pour l'hebdomadaire et ~12 pour le mensuel.

## Schema

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `report_id` | uuid | Non | - | FK `email_reports(id)` ON DELETE CASCADE. |
| `period_identifier` | text | Non | - | Période visée (`2026-W35`, `2026-08`). |
| `status` | text | Non | `'queued'` | Voir *Statuts*. |
| `attempts` | integer | Non | 0 | Nombre de rendus tentés sur cette période. |
| `last_error` | text | Oui | - | Message d'erreur de la dernière tentative. |
| `updated_at` | timestamptz | Non | now() | Maintenu par `trg_report_video_renders_updated_at`. |

PK composite `(report_id, period_identifier)`. Index partiel `idx_rvr_status` sur les statuts actifs.

## Statuts

| Statut | Sens |
|--------|------|
| `queued` | À rendre. Non utilisé à date : la route passe directement à `rendering`. |
| `rendering` | Sandbox en cours. **Load-bearing** : empêche le retry de 04:00 de lancer un second rendu en parallèle du rendu de 02:00. Au-delà de 30 min, le sandbox est considéré mort et une reprise est autorisée. |
| `ready` | MP4 disponible dans le bucket `report-videos`. |
| `error` | Échec ; `last_error` porte le détail, repris tel quel dans l'e-mail d'alerte et dans la carte « Vidéo » de `/reports/[key]`. |
| `expired` | Fichier purgé après la rétention de 7 jours. La ligne survit au fichier. Une demande de re-rendu sur une ligne `expired` repart en tentative 1. |

## RLS

| Opération | Politique |
|-----------|-----------|
| SELECT | `report_video_renders_admin_select` : `profiles.role = 'admin'`. |
| INSERT / UPDATE / DELETE | **Aucune policy.** La table est alimentée exclusivement en `service_role` par le renderer, comme `email_report_runs`. |

## Chaîne d'exécution

```
02:00  purge des vidéos de plus de 7 jours, puis rendu de la période qui bascule
04:00  nouvelle tentative si status = 'error' ou ligne absente
05:30  watchdog : alerte si toujours pas 'ready'
       (traiter aussi 'rendering' + updated_at > 30 min comme un échec)
07:00  send-email-reports attache la vidéo si status = 'ready'
```

Si la vidéo manque à 07:00, le rapport **part quand même**, avec un encart qui explique pourquoi, et le run est marqué `partial`.

## Voir aussi

- [`get_report_video_variables`](../functions/get_report_video_variables.md) - payload du template.
- [`purge_expired_report_videos`](../functions/purge_expired_report_videos.md) - rétention 7 jours.
- [`email_reports`](./email_reports.md) - l'opt-in vit dans `options.video`.
