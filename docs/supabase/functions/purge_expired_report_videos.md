# Function: purge_expired_report_videos

Rétention des vidéos de classement : supprime les MP4 de plus de N jours du bucket `report-videos` et bascule les lignes `report_video_renders` correspondantes en `expired`.

Introduite par la migration **082 (28/08/2026)**.

## Signature

```sql
purge_expired_report_videos(
  p_retention_days INTEGER DEFAULT 7,
  p_dry_run        BOOLEAN DEFAULT FALSE
) RETURNS JSONB
```

`SECURITY DEFINER`. Révoquée à `PUBLIC`, `anon` et `authenticated`. Appelée par le rendu de 02:00, avant de produire la vidéo du jour.

## Pourquoi une purge explicite

**Supabase Storage n'expose pas de règle de cycle de vie**, alors que S3 le supporte en dessous : c'est une demande de fonctionnalité toujours ouverte côté Supabase. Il n'y a donc pas de case à cocher sur le bucket, la purge doit être programmée.

La suppression passe par `gdpr_storage_delete()`, le helper existant qui appelle l'API Storage via `pg_net`. Un `DELETE FROM storage.objects` supprimerait la ligne mais **laisserait un orphelin dans le bucket S3**.

## Pourquoi 7 jours suffisent

La vidéo **voyage dans l'e-mail** en pièce jointe, elle n'est pas liée. La purger ensuite ne casse donc aucun message déjà envoyé. Avec un lien hébergé, une rétention courte aurait cassé tous les e-mails passés : les deux décisions sont cohérentes entre elles.

Conséquence pratique : la publication sur les réseaux doit se faire dans la semaine. Au-delà, il faut relancer un rendu depuis `/reports/[key]`, ce qui prend un clic.

Bénéfice RGPD : une durée de conservation courte et automatique sur des fichiers portant pseudonymes et avatars est un argument dans la bonne colonne.

## Bascule en `expired`

Les lignes `ready` dont `updated_at` dépasse la rétention passent à `expired`. Sans ça, la carte « Vidéo » de `/reports/[key]` continuerait de proposer un aperçu vers un fichier disparu.

Côté renderer, une ligne `expired` est traitée comme absente : une demande de re-rendu repart en tentative 1, même si `attempts` valait 2.

## Retour

```json
{
  "dry_run": false,
  "retention_days": 7,
  "files": 3,
  "executed_at": "2026-08-28T02:00:00Z"
}
```

`p_dry_run := true` compte les fichiers concernés sans rien supprimer ni modifier.

## Voir aussi

- [`report_video_renders`](../tables/report_video_renders.md)
- [`gdpr_storage_delete`](./gdpr_purge_orphan_storage.md) - helper de suppression réelle.
