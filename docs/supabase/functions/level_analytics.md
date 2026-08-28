# Functions: statistiques de niveaux (`/analytics/levels`)

Quatre RPC admin-only pour la page admin `/analytics/levels` :
`get_level_summary`, `get_level_stats` et `get_level_average_timeline`
(**migration 084**), plus `get_level_members` et le coût des paliers
(**migration 085**, corrigée par la 085b).

## Le modèle en un paragraphe

Le niveau d'un Compagnon **n'est pas stocké**. Il dérive de son XP de la saison
courante (année calendaire) via `get_season_xp` + `compute_level_from_xp`, et
seul le `cashback_coefficient` en est le reflet persisté (trigger sur `gains`).

Conséquence directe : **il n'existe aucun historique de passage de niveau**. Les
durées de palier exposées ici sont **reconstituées** en rejouant le cumul
chronologique de `gains.xp` par joueur, puis en datant le premier gain qui fait
franchir chaque `level_thresholds.xp_required`. C'est exact tant que l'historique
des `gains` n'est pas altéré ; une suppression de ticket (`admin_delete_receipt`)
ou une correction `rollback_beta_correction` réécrit donc rétroactivement ces durées.

## Périmètre commun

Les trois fonctions ne comptent que les **clients réels** :
`role = 'client'`, `is_test = false`, `deleted_at IS NULL`. Les comptes du
personnel gagnent aussi de l'XP (≈ 1 800 XP cumulés sur 15 comptes en 2026) mais
ne sont pas la communauté mesurée.

Le paramètre `p_year` est optionnel : `NULL` (défaut) vise la saison en cours.
Les bornes sont `[1er janvier, 1er janvier suivant)` dans le fuseau de la base,
comme `get_season_xp` — ce n'est pas Europe/Paris, mais la cohérence avec le
calcul de niveau du front prime sur la cohérence avec les RPC analytics.

## `get_level_summary(p_year integer DEFAULT NULL)`

Une seule ligne, destinée aux StatCards.

| Champ | Description |
|-------|-------------|
| `season_year`, `season_start`, `season_end` | Saison résolue |
| `days_elapsed`, `days_remaining` | Avancement de la saison (`days_remaining = 0` sur une saison close) |
| `clients_total` | Tous les clients réels, **y compris ceux à 0 XP** |
| `players_with_xp` / `players_without_xp` | Répartition du précédent |
| `avg_xp`, `median_xp`, `max_xp` | XP de saison des joueurs actifs |
| `avg_level`, `max_level` | Niveau moyen et niveau du meilleur joueur |
| `projected_max_level` | Niveau du meilleur joueur **projeté au 31/12** au rythme observé |
| `inactive_30d` | Joueurs sans gain d'XP depuis > 30 jours |
| `top_level_available` | Dernier niveau existant dans `level_thresholds` |

L'écart entre `projected_max_level` et `top_level_available` mesure la part de la
grille **hors de portée** sur la saison. La page en fait un bandeau d'alerte.

## `get_level_stats(p_year integer DEFAULT NULL)`

Une ligne par niveau de `level_thresholds`, **y compris les niveaux vides**.

| Champ | Description |
|-------|-------------|
| `level`, `level_name`, `xp_required`, `next_xp_required` | Palier (`next_xp_required` NULL sur le dernier niveau) |
| `users_count` | Joueurs **actuellement** à ce niveau |
| `reached_count` | Joueurs dont le niveau courant est ≥ à celui-ci → entonnoir. Le taux de passage se calcule côté UI : `reached(N) / reached(N−1)` |
| `transitions_count` | Franchissements observés du palier précédent vers celui-ci |
| `avg_days_to_level`, `median_days_to_level` | Durée du palier. Point d'ancrage du niveau 2 : `GREATEST(profiles.created_at, début de saison)` |
| `avg_progress_pct` | Progression moyenne dans le palier, en % du suivant |
| `near_next_count` | Joueurs à ≥ 80 % du niveau suivant (cible de relance) |
| `inactive_30d_count`, `avg_days_since_last_xp` | Stagnation |
| `avg_cashback_coefficient` | Coefficient PdB moyen (déterministe : `1 + (level−1) × 0,2`) |
| `pdb_generated_cents` | **Tous** PdB générés sur la saison (organiques + quêtes + bonus + classement) |
| `receipts_count`, `euro_spent_cents` | Activité de saison ; euros = `receipt_lines` `card` + `cash` |
| `projected_users_count` | Effectif projeté à ce niveau au 31/12 |
| `theoretical_euro_cents` | **Coût théorique du palier** : XP à combler ÷ `constants.xp_gains`, en euros. NULL au niveau 1 |
| `median_euro_to_level_cents`, `avg_euro_to_level_cents` | Euros **réellement** dépensés pendant le palier |
| `median_pdb_to_level_cents` | PdB gagnés pendant le palier, toutes sources |
| `median_receipts_to_level` | Passages en caisse pendant le palier |
| `median_euro_total_cents` | Euros cumulés **depuis l'inscription** jusqu'au franchissement de ce niveau. C'est la médiane des cumuls, **pas** la somme des médianes de paliers |

### Fenêtre du palier (migration 085)

Le coût d'un palier se mesure sur `(franchissement précédent, franchissement de
ce niveau]` — borne basse **exclusive** : le gain qui a déclenché le niveau
précédent appartient au palier précédent. Au premier franchissement il n'y a pas
de gain déclencheur, la borne recule d'une microseconde sous l'ancre
(`GREATEST(profiles.created_at, début de saison)`) pour l'inclure.

Deux conséquences à connaître :

- Un gain qui fait **sauter deux niveaux d'un coup** donne une fenêtre vide au
  second : 0 €, 0 PdB, 0 j. C'est correct — ce palier n'a rien coûté de plus.
- **Seuls `card` et `cash` comptent.** `create_receipt` →
  `validate_payment_methods` n'alimente `amount_for_gains` que depuis ces deux
  moyens de paiement : un règlement en PdB ne rapporte aucun XP. L'assiette euros
  est donc exactement celle qui fait monter de niveau.

Un coût réel **inférieur** au théorique signifie que les quêtes et bonus ont
apporté de l'XP hors achat. Un coût **supérieur** vient de l'arrondi par ticket
et du dépassement du seuil par le gain déclencheur.

## `get_level_members(p_level integer, p_year integer DEFAULT NULL)`

Les joueurs **actuellement** au niveau demandé, triés par XP décroissant.
Volume borné par l'effectif du palier (≈ 240 max à date) : pas de pagination
serveur, l'UI pagine côté client.

| Champ | Description |
|-------|-------------|
| `customer_id`, `pseudo` | `username`, sinon prénom + nom, sinon « Sans pseudo ». Pas d'e-mail : la fiche `/users/[id]` est à un clic |
| `season_xp`, `xp_to_next`, `progress_pct` | Position dans le palier (`NULL` au dernier niveau) |
| `reached_at`, `days_at_level` | Arrivée au niveau. `reached_at` est **NULL au niveau 1** (jamais franchi) ; `days_at_level` retombe alors sur l'ancienneté du compte |
| `last_xp_at`, `days_since_last_xp` | Dernière activité |
| `receipts_count`, `euro_spent_cents`, `pdb_generated_cents` | **Toute la saison**, pas seulement le palier en cours |
| `projected_level` | Niveau projeté au 31/12 à son rythme |

⚠️ **Biais de survie sur les durées** : seuls les joueurs ayant *effectivement*
franchi le palier sont comptés. Ceux qui y stagnent encore ne le sont pas, donc
la durée réelle typique est **plus longue** que la médiane affichée. Lire
`median_days_to_level` avec `transitions_count` et `users_count`.

⚠️ **Coût et valeur rattachés au niveau ACTUEL** : `pdb_generated_cents`,
`receipts_count` et `euro_spent_cents` couvrent toute la saison des joueurs qui
se trouvent aujourd'hui à ce niveau — une partie de ces dépenses a donc été
réalisée alors qu'ils étaient à un niveau inférieur, avec un coefficient plus bas.

## `get_level_average_timeline(p_year integer DEFAULT NULL)`

Une ligne par **semaine ISO** (lundi) écoulée de la saison.

| Champ | Description |
|-------|-------------|
| `week_start` | Lundi de la semaine ISO |
| `players_count` | Joueurs ayant **déjà** gagné de l'XP à cette date = dénominateur de `avg_level` |
| `avg_level`, `max_level` | Niveau moyen et maximum à la fin de cette semaine |
| `total_xp` | XP cumulé de la communauté |

Le dénominateur grossit à mesure que des joueurs entrent : les arrivants entrent
au niveau 1 et tirent la moyenne vers le bas. Une `avg_level` plate ne veut donc
pas dire que personne ne progresse — c'est à croiser avec `players_count`.

## Projection de fin de saison

Même formule dans `get_level_summary` et `get_level_stats` :

```
rythme       = season_xp / GREATEST(14, jours depuis le premier gain)
projected_xp = season_xp + rythme × jours restants dans la saison
```

Le plancher de **14 jours** évite qu'un compte créé l'avant-veille extrapole un
rythme aberrant sur dix mois. La projection est purement linéaire : elle ne
modélise ni la saisonnalité, ni l'attrition, ni l'accélération liée à la montée
du coefficient. À lire comme un ordre de grandeur, pas comme une prévision.

## Sécurité

- `SECURITY DEFINER` + `PERFORM assert_admin()` sur les trois — **admin only**
  (donc `service_role` passe aussi).
- `REVOKE FROM PUBLIC, anon` ; `GRANT EXECUTE TO authenticated, service_role`.

## Coût

Sur les volumes d'août 2026 (676 joueurs actifs, 6 500 gains XP, ~840
franchissements) : `get_level_stats` ≈ **120 ms**, `get_level_members` ≈
**145 ms**. Rien à indexer : les fenêtres travaillent sur des jeux de lignes
déjà réduits par `idx_gains_customer_id`.

## Anomalie relevée dans la grille (août 2026)

`theoretical_euro_cents` rend visible un décrochage de `level_thresholds`. Les
paliers grandissent d'un facteur très régulier (~1,16 à 1,19×) à partir du
niveau 8, **sauf au niveau 16** :

| Palier | XP | Facteur vs palier précédent | € théoriques |
|---|---:|---:|---:|
| N15 | 2 119 | 1,15× | 1 277 € |
| **N16** | **7 165** | **3,38×** | **4 316 €** |
| N17 | 4 436 | 0,62× | 2 672 € |

Le N16 est plus cher que le N17 : ce n'est pas un palier de prestige, c'est une
irrégularité (le N26, à 1,94×, en est un — le palier suivant n'existe pas).
Sans joueur au-delà du N10 à date, l'anomalie est sans effet pratique, mais elle
piquera dès qu'un joueur approchera. La grille s'édite dans `/content/storytelling`.

## Exemple

```typescript
// src/lib/services/analyticsService.ts
const summary = await getLevelSummary();      // saison courante
const stats   = await getLevelStats(2026);
const weekly  = await getLevelAverageTimeline(2026);
```
