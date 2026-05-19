# Function: check_email_exists

Vérifie si une adresse email existe dans `auth.users`. **N'est plus appelable depuis le REST** depuis la migration **040 (Security Definer hardening, 18/05/2026)**.

## Signature

```sql
CREATE FUNCTION public.check_email_exists(email_to_check TEXT) RETURNS boolean
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
```

## Parametres

| Parametre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `email_to_check` | `TEXT` | Oui | Email à vérifier (case-sensitive selon la collation `auth.users`). |

## Retour

`BOOLEAN` : `true` si une ligne `auth.users` avec cet email existe, `false` sinon.

## Securite

Depuis la migration **040 (18/05/2026)** :

- `REVOKE EXECUTE FROM PUBLIC, anon, authenticated` — **aucun GRANT** n'est ré-accordé.
- Conséquence : la fonction n'est plus appelable via PostgREST (REST/RPC). Reste accessible uniquement par le owner (`postgres`) en interne, et donc par `service_role` via la connexion superuser.

### Pourquoi cette restriction

Avant la migration 040, la fonction était appelable par `anon` (utilisateur non connecté). Ça en faisait un **oracle d'énumération d'adresses emails** : un attaquant pouvait deviner quels comptes existent en testant des emails un par un. Risque RGPD + facilitation de phishing ciblé.

### Alternative recommandée

Pour les formulaires d'inscription / login côté front :

- Laisser **Supabase Auth signUp** gérer le doublon. Le SDK lève une erreur dédiée si l'email existe déjà ; le front affiche un message générique du style « Si cette adresse a déjà un compte, vous recevrez un email ».
- Pour le flow « mot de passe oublié », même logique : appeler `signInWithOtp` / `resetPasswordForEmail` ; aucun feedback ne doit révéler si l'email existe.

## Notes

- Si une feature admin a vraiment besoin de vérifier l'existence (par ex. import en masse), passer par une Edge Function avec `service_role` plutôt que de ré-exposer cette RPC.
- Aucune autre RPC du codebase n'expose équivalent (`get_user_info` ne lit qu'à partir d'UUIDs connus).
