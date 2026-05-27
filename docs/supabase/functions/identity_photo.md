# Fonctions Photo d'identité

Gestion de la photo d'identité des utilisateurs avec cooldown de 30 jours.

## enforce_identity_photo_cooldown

Trigger function qui empêche le changement de photo d'identité plus d'une fois tous les 30 jours.

### Trigger

```sql
CREATE TRIGGER trg_identity_photo_cooldown
  BEFORE UPDATE ON profiles
  FOR EACH ROW
  EXECUTE FUNCTION enforce_identity_photo_cooldown();
```

### Logique

1. Si `identity_photo_url` n'a pas changé → passe
2. Si ancienne valeur NULL (premier upload) → autorise et set `identity_photo_updated_at = NOW()`
3. Si l'utilisateur courant est admin → autorise (bypass cooldown)
4. Si `identity_photo_updated_at` < 30 jours → **bloque** avec errcode `P0001` et message `IDENTITY_PHOTO_COOLDOWN_ACTIVE:{jours_restants}`
5. Sinon → autorise et met à jour `identity_photo_updated_at`

### Gestion d'erreur côté client

Le message d'erreur contient le nombre de jours restants après le `:`, parsable côté front pour affichage UX.

---

## admin_reset_identity_photo_cooldown

Permet à un admin de réinitialiser le cooldown d'un utilisateur.

### Signature

```sql
admin_reset_identity_photo_cooldown(p_user_id uuid) RETURNS void
```

### Paramètres

| Paramètre | Type | Description |
|-----------|------|-------------|
| `p_user_id` | `uuid` | ID de l'utilisateur dont réinitialiser le cooldown |

### Autorisation

- Admin uniquement (vérifie `profiles.role = 'admin'` pour `auth.uid()`)
- `SECURITY DEFINER`
- Errcode `P0003` si non autorisé

### Effet

Met `identity_photo_updated_at = NULL` pour l'utilisateur cible, ce qui lui permet de re-uploader immédiatement.
