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
- `SECURITY DEFINER` avec `search_path = 'public'`

### Opérations effectuées

1. **Suppression** : likes, comments, user_badges, quest_progress, quest_completion_logs, coupons non utilisés
2. **Dé-liaison** : coupon_templates.created_by, quests.created_by, period_reward_configs.created_by/distributed_by, receipts.employee_id, coupon_distribution_logs.distributed_by → NULL
3. **Anonymisation du profil** : email → `deleted-{uuid}@deleted.local`, username → `deleted-{8chars}`, prénom/nom/phone/birthdate/avatar/identity_photo → NULL, `deleted_at` → now()
4. **Traçabilité** : insère une ligne `erasure` dans `gdpr_requests` avec le détail des suppressions

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
