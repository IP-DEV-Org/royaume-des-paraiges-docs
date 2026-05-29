# Table: quests

Définition des quêtes périodiques disponibles. Modèle template récurrent : une quête est instanciée par utilisateur × période via `quest_progress`.

## Terminologie produit

La table `quests` couvre (à terme) deux réalités produit distinctes. Les noms techniques (`quests`, `quest_progress`, `quest_type`, RPC) restent inchangés ; la distinction est fonctionnelle.

| Produit | Implémentation BDD | État |
|---------|--------------------|------|
| **Défi** (quête récurrente) | `quests.period_type ∈ {weekly, monthly, yearly}` + `quest_progress` remis à zéro à chaque période | ✅ en prod |
| **Mission** (quête ponctuelle, one-shot) | À définir (modèle différé — pas encore implémenté) | ⏳ à venir |

Le mot **quête** reste acceptable dans la doc **uniquement** si on précise « quête récurrente » (défi) ou « quête ponctuelle » (mission) — seul, il est ambigu.

## Informations

| Propriete | Valeur |
|-----------|--------|
| **Schema** | `public` |
| **RLS** | Active |
| **Lignes** | 32 (avril 2026 : 7 actives, 17 doublons désactivés, 6 consumption inactives, 2 monthly extra) |

## Colonnes

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | `bigint` | Non | - | PK |
| `name` | `varchar(255)` | Non | - | Nom narratif (ex. « Cent pas sur la piste ») |
| `description` | `text` | Oui | - | Description fonctionnelle courte |
| `lore` | `text` | Oui | - | Texte narratif immersif (~150 chars), affiché dans la modale côté client |
| `slug` | `varchar(100)` | Non | - | Identifiant unique (UNIQUE) |
| `quest_type` | enum `quest_type` | Non | - | 7 valeurs (cf. ci-dessous) |
| `consumption_type` | enum `consumption_type` | Oui | - | **Obligatoire ssi `quest_type = 'consumption_count'`** (CHECK constraint). Sinon NULL. |
| `target_value` | `integer` | Non | - | Objectif à atteindre. Pour `amount_spent`, en **centimes** (50 € = 5000) — saisie côté admin en € puis ×100. Pour `cashback_earned`, directement en **PdB** (50 PdB = 50). |
| `period_type` | `varchar(20)` | Non | - | `weekly` / `monthly` / `yearly` |
| `coupon_template_id` | `bigint` | Oui | - | FK → `coupon_templates`. Récompense en bonus PdB direct ou coupon % à la complétion. |
| `badge_type_id` | `bigint` | Oui | - | FK → `badge_types`. Badge attribué à la complétion. |
| `bonus_xp` | `integer` | Non | 0 | XP bonus en plus des gains normaux |
| `bonus_cashback` | `integer` | Non | 0 | Cashback bonus en centimes |
| `is_active` | `boolean` | Non | true | Visible côté client. Désactiver pour archiver sans supprimer. |
| `display_order` | `integer` | Non | 0 | Ordre d'affichage |
| `created_at` | `timestamptz` | Non | now() | - |
| `updated_at` | `timestamptz` | Non | now() | - |
| `created_by` | `uuid` | Oui | - | Admin qui a créé la quête |

## Enum `quest_type`

| Valeur | Description |
|---|---|
| `xp_earned` | Gagner X XP sur la période |
| `amount_spent` | Dépenser X centimes sur la période. Saisie en € côté admin (×100 au submit). Ajouté à la création, mais à éviter en quête client si la règle « zéro euro côté client » s'applique — utiliser `cashback_earned` à la place. |
| `cashback_earned` | Collecter X Paraiges de Bronze (1 PdB = 1 centime) sur la période. Progression = SUM(`gains.cashback_money`), coefficient client + bonus coupons inclus. Ajouté en migration 028. |
| `establishments_visited` | Visiter X établissements distincts sur la période |
| `orders_count` | Passer X commandes sur la période |
| `quest_completed` | Compléter au moins 1 quête dans X sous-périodes (`monthly→weekly`, `yearly→monthly`). **Incompatible avec `weekly`**. |
| `consumption_count` | Consommer X produits du type `consumption_type` (avril 2026). Calcul : SUM(`receipt_consumption_items.quantity` WHERE `consumption_type = quests.consumption_type`) sur la période. |

## Enum `consumption_type` (utilisé par `consumption_count`)

`cocktail`, `biere`, `alcool`, `soft`, `boisson_chaude`, `restauration`

## Contraintes CHECK

- `quests_consumption_type_consistency` : `(quest_type = 'consumption_count' AND consumption_type IS NOT NULL) OR (quest_type <> 'consumption_count' AND consumption_type IS NULL)`

## Clés primaires

- `id`

## Index uniques

- `quests_slug_key` UNIQUE sur `slug`

## Relations (Foreign Keys)

- `quests_badge_type_id_fkey` : `badge_type_id` → `badge_types.id`
- `quests_coupon_template_id_fkey` : `coupon_template_id` → `coupon_templates.id`
- `quests_created_by_fkey` : `created_by` → `profiles.id`

## Fonctions liées

- `calculate_quest_progress(p_customer_id, p_quest_id, p_period_identifier)` — calcule la progression actuelle pour un user × quête × période. Branche par `quest_type` (étendue avec `consumption_count` en migration 011, puis `cashback_earned` en migration 029).
- `update_quest_progress_for_receipt(p_receipt_id)` — appelée par `create_receipt`, propage les progressions à toutes les quêtes actives.
- `distribute_quest_reward(p_quest_progress_id)` — attribue le coupon + badge + bonus XP/cashback à la complétion.

## Quêtes actives en prod (avril 2026, post-refonte)

7 quêtes actives pour les Compagnons :

| ID | Slug | Type | Period | Cible | Bonus | Badge |
|---|---|---|---|---|---|---|
| 8 | `gagner_100xp` | xp_earned | weekly | 100 XP | +25 XP / 5 € | — |
| 9 | `passer_10_commandes` | orders_count | weekly | 10 cmd | +25 XP / 5 € | — |
| 27 | `500xp_pour_15` | xp_earned | monthly | 500 XP | +100 XP / 15 € | — |
| 28 | `30_commandes_pour_12` | orders_count | monthly | 30 cmd | +100 XP / 15 € | — |
| 30 | `monthly_tour_royaume` | establishments_visited | monthly | 3 tavernes | +50 XP / 5 € | quest_pelerin |
| 31 | `monthly_le_bon_buveur` | cashback_earned | monthly | 50 PdB | +50 XP / 10 € | — |
| 32 | `yearly_fidele_parmi_fideles` | quest_completed | yearly | 12 mois | +500 XP / 50 € | quest_fidele_legendary |

## Quêtes consumption hebdo prêtes mais désactivées (avril 2026)

6 quêtes `consumption_count` (`is_active = false` par défaut), à activer une à une depuis l'admin :

| Slug | consumption_type | Cible |
|---|---|---|
| `weekly_5_bieres` | biere | 5 |
| `weekly_3_cocktails` | cocktail | 3 |
| `weekly_2_alcools` | alcool | 2 |
| `weekly_3_softs` | soft | 3 |
| `weekly_2_boissons_chaudes` | boisson_chaude | 2 |
| `weekly_1_restauration` | restauration | 1 |

## Scoping par établissement (M2M via `quests_establishments`)

Depuis la migration 020 (avril 2026), une quête peut être scopée à un sous-ensemble d'établissements via la table [`quests_establishments`](./quests_establishments.md) :

- **Aucune entrée** pour une quête = **quête globale** (applicable partout). C'est le défaut actuel pour toutes les quêtes.
- **≥ 1 entrée** = **quête locale** (restreinte aux établissements listés).

## Prévention des quêtes redondantes

L'incident d'avril 2026 (multi-complétion sur un seul ticket, ratio PdB/ticket jusqu'à 241 %) a été causé par 14 quêtes hebdo avec seuils chevauchants (ex. 6 variantes `xp_earned weekly` de 60 à 200 XP). Le plan de prévention en 4 PR :

- **PR#1** ✅ (migration 020) — prérequis structurels : `quests_establishments`, `admin_settings`, vue `avg_ticket_12m`.
- **PR#2** ✅ (migration 021) — triggers PL/pgSQL `trg_quests_enforce_redundancy`, `trg_qe_enforce_redundancy_insert`, `trg_qe_enforce_redundancy_delete` s'appuyant sur la fonction [`check_quest_redundancy`](../functions/check_quest_redundancy.md). Bloquent toute activation ou modification de scope qui créerait une redondance signature-équivalente. Exception `P0421` avec DETAIL JSON parseable.
- **PR#3** ✅ UX admin : picker M2M d'établissements (`components/establishments-picker.tsx`), interception des exceptions `P0421` via `lib/supabase/errorParser.ts` et modal `components/quest-conflict-dialog.tsx` sur les pages `/quests/create`, `/quests/[id]` et `/quests` (toggle d'activation).
- **PR#4** ✅ Dashboard `/quests/health` (ratio bonus PdB / panier attendu + alertes) et page `/settings` (CRUD des clés `quest_alert_ratio_pct` et `quest_reference_prices_cents`). Service `lib/services/adminSettingsService.ts`.

Bonus standard : +25 XP / 2-5 €. Pas de badge attaché en V1.
