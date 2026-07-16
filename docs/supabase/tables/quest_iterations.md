# Table: quest_iterations

Overrides par itération des **quêtes répétables** (migration 066, juillet 2026). Une quête `is_repeatable = true` (**opt-in** — défaut `false` depuis la migration 067) peut être complétée plusieurs fois par période selon le niveau du joueur (barème `admin_settings['quest_repeat_level_tiers']`) ; cette table permet de personnaliser l'objectif et les gains de chaque itération.

Sémantique :

- **L'itération 1 = la quête elle-même** (`quests.target_value`, `coupon_template_id`, `bonus_xp`, `bonus_cashback`). Les lignes de cette table couvrent les itérations **≥ 2** (CHECK).
- **Le nombre de lignes borne le plafond de complétions** (migration 069) : `plafond effectif = LEAST(plafond du rang, 1 + nb de lignes)`. Une quête `is_repeatable` sans aucune ligne = 1 seule complétion ; pour ouvrir N complétions, l'admin ajoute N−1 itérations. Le front applique la même borne (`buildQuestWithProgress`).
- **Chaque champ NULL hérite de la quête de base** (granularité par champ : on peut surcharger seulement l'objectif, seulement le bonus XP, etc.).
- Une itération **sans ligne** hérite intégralement de la base.
- Limitation assumée : impossible d'exprimer « pas de coupon à l'itération k alors que la base en a un » (NULL = héritage). Les bonus XP/PdB, eux, peuvent être explicitement mis à `0`.

## Colonnes

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `quest_id` | `bigint` | Non | - | FK → `quests.id` (ON DELETE CASCADE). PK composite avec `iteration`. |
| `iteration` | `integer` | Non | - | Numéro d'itération, CHECK `>= 2`. |
| `target_value` | `integer` | Oui | - | Objectif de CETTE itération (CHECK `> 0`). NULL = hérite de `quests.target_value`. Le seuil de déblocage de l'itération k = somme des objectifs des itérations 1..k (cumul continu). |
| `coupon_template_id` | `bigint` | Oui | - | FK → `coupon_templates` (ON DELETE SET NULL). NULL = hérite du coupon de base. |
| `bonus_xp` | `integer` | Oui | - | CHECK `>= 0`. NULL = hérite. `0` = pas de bonus XP à cette itération. |
| `bonus_cashback` | `integer` | Oui | - | Centimes/PdB. CHECK `>= 0`. NULL = hérite. `0` = pas de bonus PdB à cette itération. |
| `created_at` | `timestamptz` | Non | now() | - |
| `updated_at` | `timestamptz` | Non | now() | Trigger `set_quest_iterations_updated_at`. |

## Clé primaire

- (`quest_id`, `iteration`)

## RLS

| Policy | Action | Condition |
|---|---|---|
| `Authenticated users can view quest iterations` | SELECT | `true` (le front lit les paliers pour afficher seuils et gains par itération) |
| `Admin full access on quest_iterations` | ALL | `profiles.role = 'admin'` |

## Consommateurs

- **Trigger `distribute_quest_rewards`** (via `get_quest_iteration_target()` + lecture directe) : résolution de l'objectif et des gains de chaque itération à la distribution.
- **Admin** : section « Itérations supplémentaires » du formulaire de quête (`/quests/create`, `/quests/[id]`), persistée par `setQuestIterations()` (pattern DELETE + INSERT).
- **Front Expo** : jointure dans les selects de `QuestService` pour calculer les seuils cumulés et afficher les gains de l'itération courante.
