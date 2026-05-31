# Table: quest_periods

Table de liaison pour assigner des quêtes à des périodes spécifiques (ex: 2026-W05, 2026-01, 2026)

## Informations

| Propriete | Valeur |
|-----------|--------|
| **Schema** | `public` |
| **RLS** | Active |
| **Lignes** | 0 |

## Colonnes

| Colonne | Type | Nullable | Default | Description |
|---------|------|----------|---------|-------------|
| `id` | `bigint` | Non | - | - |
| `quest_id` | `bigint` | Non | - | - |
| `period_identifier` | `character varying(20)` | Non | - | Identifiant de période au format YYYY-Www (weekly), YYYY-MM (monthly), ou YYYY (yearly) |
| `created_at` | `timestamp with time zone` | Oui | now() | - |

## Cles primaires

- `id`

## Relations (Foreign Keys)

- `quest_periods_quest_id_fkey`: quest_id → quests.id

## Sémantique : planning appliqué par le moteur

Depuis la migration `041_quest_progress_respect_periods`, le planning n'est plus
purement cosmétique côté admin : **le moteur de récompense l'applique**.

- Une quête **sans entrée** `quest_periods` est **permanente** : elle progresse et
  récompense à **chaque** période où elle est `is_active = true`.
- Une quête **avec** des `quest_periods` ne progresse/récompense **que** si la
  période courante (`get_period_identifier(quest.period_type)`) figure dans son
  planning. C'est ainsi qu'on implémente la **rotation** (ex. quêtes hebdo de
  consommation actives 1 semaine sur 3).

La règle est appliquée à l'identique par `update_quest_progress_for_receipt`
(progression temps réel sur chaque receipt) et `update_meta_quest_progress`
(méta-quêtes `quest_completed`).

> ⚠️ Avant la migration 041, `update_quest_progress_for_receipt` ignorait
> `quest_periods` : toute quête `is_active` récompensait chaque période. Cela a
> causé des crédits PdB hors planning (rotation court-circuitée) et le maintien
> en paiement de quêtes défuntes restées actives sans période. La page
> `/quests/health` (admin) signale désormais les deux configurations à risque :
> quête active **permanente** (aucune période) et quête active à **planning
> périmé** (aucune période ≥ période courante).
