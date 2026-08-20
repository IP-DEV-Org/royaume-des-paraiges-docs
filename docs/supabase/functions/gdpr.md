# Fonctions RGPD

Ensemble de fonctions pour la conformité RGPD : export de données, anonymisation et purge automatique.

## gdpr_export_user_data

Exporte l'intégralité des données personnelles d'un utilisateur au format JSON.

### Signature

```sql
gdpr_export_user_data(target_user_id uuid) RETURNS jsonb
```

### Paramètres

| Paramètre | Type | Description |
|-----------|------|-------------|
| `target_user_id` | `uuid` | ID de l'utilisateur dont exporter les données |

### Autorisation

- L'utilisateur peut exporter ses propres données
- Un admin peut exporter les données de n'importe quel utilisateur
- `SECURITY DEFINER` avec `search_path = 'public'`

### Données exportées

`profile`, `receipts`, `gains`, `spendings`, `coupons`, `likes`, `comments`, `badges`, `quest_progress`, `leaderboard_rewards`

### Effet de bord

Insère une ligne `request_type = 'export'` dans `gdpr_requests`.

---

## gdpr_anonymize_user

Anonymise un utilisateur : supprime les données personnelles, neutralise le profil, conserve les données transactionnelles pour comptabilité.

### Signature

```sql
gdpr_anonymize_user(target_user_id uuid) RETURNS jsonb
```

### Paramètres

| Paramètre | Type | Description |
|-----------|------|-------------|
| `target_user_id` | `uuid` | ID de l'utilisateur à anonymiser |

### Autorisation

- L'utilisateur peut demander sa propre suppression
- Un admin peut anonymiser n'importe quel utilisateur
- **La cible doit être un compte `client`** (migration 073) : un compte `employee` / `establishment` / `admin` lève `P0424` (`ACCOUNT_ROLE_PROTECTED:`), y compris pour l'intéressé lui-même et pour un admin. Le refus intervient **avant** l'écriture dans `gdpr_requests`. Pour supprimer un compte du personnel, le repasser d'abord en `client`. Garde-fou doublé côté triggers `BEFORE DELETE` (`auth.users` + `profiles`), cf. [triggers](../triggers/README.md#trg_protect_staff_account_delete--trg_protect_staff_profile_delete-migration-073).
- `SECURITY DEFINER` avec `search_path = 'public'`

### Opérations effectuées

1. **Suppression** : likes, comments, user_badges, quest_progress, quest_completion_logs, coupons non utilisés
2. **Dé-liaison** : coupon_templates.created_by, quests.created_by, period_reward_configs.created_by/distributed_by, receipts.employee_id, coupon_distribution_logs.distributed_by → NULL
3. **Anonymisation du profil** : email → `deleted-{uuid}@deleted.local`, username → `deleted-{8chars}`, prénom/nom/phone/birthdate/avatar/identity_photo → NULL, `deleted_at` → now()
4. **Purge des fichiers** (migration 074) : avatar(s) et photo(s) d'identification effacés du Storage via `gdpr_purge_user_storage()`. Best-effort et asynchrone : un échec n'interrompt pas la suppression du compte, le balayage quotidien rattrape.
5. **Traçabilité** : insère une ligne `erasure` dans `gdpr_requests` avec le détail des suppressions

### Retour

```json
{
  "success": true,
  "user_id": "...",
  "profile_anonymized": true,
  "details": {
    "likes_deleted": 5,
    "comments_deleted": 2,
    "badges_deleted": 3,
    ...
  }
}
```

---

## gdpr_enforce_retention

Purge automatique selon les durées de conservation légales. Prévue pour exécution par pg_cron.

### Signature

```sql
gdpr_enforce_retention() RETURNS jsonb
```

### Autorisation

`SECURITY DEFINER` — à appeler uniquement via pg_cron ou manuellement par un admin.

### Opérations

1. **Anonymisation des comptes inactifs** (>3 ans sans activité) — clients uniquement, vérifie : aucun receipt, aucun gain, profile non modifié, et compte créé il y a >3 ans
2. **Purge des quest_completion_logs** >3 ans
3. **Purge des coupon_distribution_logs** >10 ans (obligation comptable)

---

## Purge du Storage (migration 074)

Les buckets `avatars` et `identity-photos` sont **publics** et les noms de fichiers dérivent de l'UUID (`{uuid}-{timestamp}.{ext}`) : un fichier laissé en place après suppression de compte reste accessible par URL devinable. Avant la migration 074, `handle_user_delete()` mettait les colonnes à NULL sans jamais supprimer les fichiers : 57 fichiers orphelins (dont 19 selfies d'identification) ont été purgés rétroactivement le 14/08/2026.

> ⚠️ La suppression passe par l'**API Storage** (`DELETE /storage/v1/object/{bucket}`, corps `{"prefixes": [...]}`), jamais par `DELETE FROM storage.objects`. Le DELETE SQL ne retire que la ligne de métadonnées : le blob reste dans le bucket S3, invisible mais présent. Seule l'API supprime les deux.

### gdpr_storage_delete

```sql
gdpr_storage_delete(p_bucket text, p_names text[]) RETURNS bigint
```

Primitive interne. Appelle l'API Storage via `pg_net` avec la clé service du vault (`reconcile_service_key`, même schéma que les crons `cashpad-*`). Retourne l'id de requête pg_net. Asynchrone : la requête part au COMMIT, une transaction annulée n'envoie rien. Clé absente → `WARNING` et abandon silencieux, jamais d'exception.

### gdpr_purge_user_storage

```sql
gdpr_purge_user_storage(target_user_id uuid) RETURNS int
```

Purge les fichiers d'un utilisateur dans `avatars` + `identity-photos`. Appariement par préfixe de nom **et** par `owner` (l'un ou l'autre manque sur les fichiers importés en mai 2026). Appelée par `handle_user_delete()` sous `EXCEPTION` : un incident Storage ne doit jamais faire échouer un effacement de compte. Retourne le nombre de fichiers programmés.

### gdpr_purge_orphan_storage

```sql
gdpr_purge_orphan_storage(p_dry_run boolean DEFAULT true, p_limit int DEFAULT 500) RETURNS jsonb
```

Filet de sécurité, planifié par le cron **`gdpr-purge-orphan-storage`** (`15 4 * * *`). Purge trois cas : profil avec `deleted_at IS NOT NULL`, UUID ne correspondant à aucun profil, nom sans UUID exploitable dans un bucket personnel. Un fichier encore référencé par l'`avatar_url` / `identity_photo_url` d'un compte **vivant** n'est jamais candidat.

`p_dry_run = true` par défaut : un appel nu recense sans supprimer. Ne cible que les deux buckets personnels : `content-assets` et `website` ne sont jamais touchés.

```json
{ "dry_run": false, "files": 56, "details": { "avatars": 37, "identity-photos": 19 }, "executed_at": "..." }
```

### gdpr_purge_stale_profile_media (migration 075)

```sql
gdpr_purge_stale_profile_media(p_dry_run boolean DEFAULT true, p_limit int DEFAULT 1000) RETURNS jsonb
```

Purge les **versions périmées** : tout objet des buckets personnels qu'aucun profil ne référence. À distinguer de `gdpr_purge_orphan_storage`, qui vise les fichiers des comptes *supprimés* ; ici les comptes sont vivants et on ne garde que le fichier courant.

⚠️ Garde-fou : seuls les objets créés il y a **plus d'une heure** sont candidats. Le front écrit le fichier puis le profil ; sans ce délai, un passage au mauvais moment détruirait un upload pas encore référencé.

Planifiée avec `gdpr_purge_orphan_storage` dans le cron `gdpr-purge-orphan-storage`.

### trg_purge_replaced_profile_media (migration 075)

Trigger `AFTER UPDATE OF avatar_url, identity_photo_url ON profiles` : supprime l'ancien fichier une fois le nouveau chemin committé. La purge est donc une conséquence de l'écriture en base, jamais une étape client, ce qui couvre l'app, le dashboard admin et tout appel REST direct. Sous `EXCEPTION` : un incident Storage ne fait jamais échouer la mise à jour du profil.
