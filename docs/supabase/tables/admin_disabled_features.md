# Table: admin_disabled_features

## Description

Accès par fonctionnalité entre administrateurs du dashboard admin. Une ligne = une fonctionnalité **désactivée** pour un admin ; l'absence de ligne = accès (défaut = tout activé). Seul un **super admin** (`profiles.is_super_admin = true`) peut écrire dans cette table — un admin restreint ne peut donc pas se dé-restreindre lui-même.

Les `feature_key` correspondent aux entrées de la sidebar admin et sont définies côté front dans `src/lib/features.ts` (`FEATURE_KEYS`, ~17 clés : `analytics`, `reconciliation`, `users`, `receipts`, `coupons`, `cashback-gains`, `history`, `quests`, `rewards`, `achievements`, `storytelling`, `templates`, `beers`, `establishments`, `gdpr`, `documentation`, `settings`). Pas de CHECK en BDD : la liste évolue côté front, la validation se fait par Zod à l'écriture (`toggleFeatureSchema`).

Introduite par la migration **057 (juillet 2026)**.

## Schema

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `profile_id` | uuid | Non | - | FK vers `profiles(id)` ON DELETE CASCADE. Admin restreint. |
| `feature_key` | text | Non | - | Clé de la fonctionnalité désactivée (cf. `src/lib/features.ts`). |
| `created_at` | timestamptz | Non | now() | Date de la restriction. |
| `created_by` | uuid | Oui | - | FK vers `profiles(id)` ON DELETE SET NULL. Super admin auteur. |

## Clé primaire

- `(profile_id, feature_key)`

## RLS

RLS active : **Oui**

| Policy | Action | Condition |
|---|---|---|
| `read own or super admin` | SELECT | `profile_id = auth.uid() OR is_super_admin()` |
| `super admin insert` | INSERT | `is_super_admin() AND NOT is_super_admin(profile_id)` |
| `super admin delete` | DELETE | `is_super_admin()` |

Pas de policy UPDATE : la table est insert/delete only. Le `WITH CHECK` de l'INSERT interdit aussi de restreindre un profil super admin (y compris soi-même).

GRANT : `SELECT, INSERT, DELETE` à `authenticated` (la RLS porte la sécurité), `ALL` à `service_role`, `REVOKE ALL` pour `PUBLIC` et `anon`.

## Consommateurs

- **Middleware admin** (`src/lib/supabase/middleware.ts`) : une seule query étendue `profiles.select("role, is_super_admin, admin_disabled_features!profile_id(feature_key)")` — blocage dur (redirect `/?error=feature_disabled`) des URLs des fonctionnalités désactivées. ⚠️ le `!profile_id` est obligatoire (2 FK vers `profiles`, embed ambigu sinon).
- **`CurrentAdminProvider`** (client) : masque les entrées correspondantes de la sidebar et de la palette Cmd+K.
- **Page `/settings/access`** (super admin uniquement) : matrice d'interrupteurs admins × fonctionnalités. Service : `src/lib/services/adminAccessService.ts`.

## Voir aussi

- [`is_super_admin`](../functions/is_super_admin.md) — helper d'autorisation utilisé par les policies
- [`profiles`](./profiles.md) — colonne `is_super_admin` + trigger anti-escalade `trg_protect_is_super_admin`
