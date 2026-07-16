# Table: admin_settings

## Description

Table clé-valeur (JSONB) pour le paramétrage admin. Permet d'exposer des valeurs éditables depuis l'admin sans migration SQL. Lecture/écriture admin-only.

Introduite par la migration 020 (avril 2026) dans le cadre du plan de prévention des quêtes redondantes (PR#1). Les 2 seeds initiaux concernent le check de ratio bonus/panier attendu sur les quêtes.

## Schema

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `key` | text | Non | - | PK. Identifiant unique du paramètre (snake_case). |
| `value` | jsonb | Non | - | Valeur JSONB (scalaire, objet, array) selon la clé. |
| `description` | text | Oui | - | Description du paramètre, affichée dans l'UI admin. |
| `updated_at` | timestamptz | Non | now() | Horodatage de dernière modification. |
| `updated_by` | uuid | Oui | - | FK vers `profiles(id)`. Admin qui a modifié la valeur. |

## Clé primaire

- `key`

## RLS

RLS active : **Oui**

| Policy | Action | Condition |
|---|---|---|
| `Admins can read admin settings` | SELECT | `profiles.role = 'admin'` |
| `Admins can write admin settings` | ALL | `profiles.role = 'admin'` |

Un utilisateur non-admin ne peut ni lire ni écrire dans cette table.

## Clés actuelles

| Clé | Type JSONB | Valeur | Description |
|---|---|---|---|
| `quest_alert_ratio_pct` | number | `10` | Seuil (%) : alerte sur le dashboard santé des quêtes si `bonus_cashback > (ratio / 100) × montant_reference_attendu`. |
| `quest_reference_prices_cents` | object | `{"biere": 600, "cocktail": 800, "alcool": 800, "soft": 350, "boisson_chaude": 250, "restauration": 700}` | Prix de référence par `consumption_type` (centimes), utilisés pour calculer le montant attendu à dépenser pour compléter une quête `consumption_count`. |
| `quest_repeat_level_tiers` | string ou array | `"auto"` | **Migrations 066 + 068 (bimodale).** Répétition des défis répétables : `"auto"` (défaut depuis la 068) = plafond lié au **rang** du joueur (rang N → N complétions, suit dynamiquement la table `ranks`) ; tableau `[{min_level, max_completions}]` non vide = barème **manuel** par niveau (palier de plus haut `min_level` atteint). Clé absente/invalide ⇒ mode auto ; exception ⇒ 1. Lu par la fonction SQL `get_max_quest_completions()`. Édité via la Card « Répétition des défis » de `/settings` (toggle auto/manuel). |

## Convention JSONB

Chaque clé a un **type JSONB attendu et stable**. L'UI admin (à venir PR#4) doit valider ce type avant écriture. Les types actuellement utilisés :
- `number` : scalaires (seuils, pourcentages)
- `object` : dictionnaires de sous-valeurs (ex. prix par type)

Toute évolution d'un type JSONB d'une clé existante doit passer par une migration SQL documentée.

## Exemples de requêtes

```sql
-- Lire le seuil de ratio
SELECT value::int AS ratio_pct FROM admin_settings
WHERE key = 'quest_alert_ratio_pct';

-- Lire le prix de référence d'un type de consommation
SELECT (value->>'biere')::int AS biere_cents FROM admin_settings
WHERE key = 'quest_reference_prices_cents';

-- Mettre à jour le seuil (admin uniquement)
UPDATE admin_settings
SET value = '15'::jsonb, updated_at = NOW(), updated_by = auth.uid()
WHERE key = 'quest_alert_ratio_pct';
```

## Statistiques

- Lignes : **2** (post-migration 020)
