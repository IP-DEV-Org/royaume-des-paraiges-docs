# Function: get_report_video_variables

Point d'entrée unique du **renderer vidéo** (`royaume-video-renderer`). Renvoie les variables à plat du template HyperFrames du rapport (classement **ou** défis de la période), plus la liste des avatars à télécharger.

Introduite par la migration **082 (28/08/2026)**, étendue aux défis par la migration **110 (09/09/2026)**.

## Signature

```sql
get_report_video_variables(
  p_report_key        TEXT,
  p_period_identifier TEXT DEFAULT NULL
) RETURNS JSONB
```

`STABLE`, `SECURITY DEFINER`, `assert_admin()` en tête (donc `service_role` passe). Révoquée à `PUBLIC` et `anon`.

## Pourquoi une RPC dédiée

Plutôt que de laisser le renderer consommer `get_email_report_payload` et faire le mapping lui-même :

1. **La règle d'opposition est appliquée ici, en SQL, une seule fois.** C'est une règle juridique : elle ne doit pas pouvoir être oubliée par un consommateur JavaScript.
2. L'aplatissement rang par rang est une contrainte du **moteur de rendu**, pas une forme de rapport : HyperFrames n'accepte que `string`, `number`, `color`, `boolean`, `enum`. **Il n'existe pas de type tableau**, donc le top 10 ne peut pas être une seule variable.
3. Le renderer n'a ainsi aucune logique métier : il reçoit, il rend.

## Types de rapport acceptés

| `report_type` | Template du renderer | Variables | Avatars |
|---|---|---|---|
| `leaderboard` | `templates/leaderboard` | 34 | jusqu'à 10 à télécharger |
| `new_quests` | `templates/quests` | 76 | aucun |

Tout autre type lève `P0425 REPORT_VIDEO_UNSUPPORTED`. Le renderer choisit le dossier de template d'après `email_reports.report_type`, la forme de la réponse est la même dans les deux cas.

## Résolution de la période

Identique à `get_email_report_payload` : sans `p_period_identifier`, la portée du rapport tranche (`period_scope`). Les deux rapports de classement étant en `previous`, un appel sans identifiant le lundi à 02:00 vise la semaine **close**. Les deux rapports de défis sont en `current` : le même appel vise la semaine qui **s'ouvre**, comme l'e-mail d'annonce.

⚠️ **Le renderer doit laisser ce paramètre à NULL** et ne jamais calculer un numéro de semaine côté JavaScript. La convention ISO 8601 vit en SQL. C'est exactement le défaut qui a fait distribuer `distribute_period_rewards_v2` dans le vide pendant quatre mois (cf. migration 081).

## Retour

```json
{
  "report_key": "weekly_leaderboard",
  "period_identifier": "2026-W34",
  "variables": {
    "period_label": "Semaine 34",
    "period_subtitle": "semaine du 17 au 23 août 2026",
    "participants": 132,
    "total_xp": 9380,
    "rank1_name": "vitaline",
    "rank1_xp": 395,
    "rank1_avatar": "assets/avatars/rank1.jpg"
  },
  "avatars": [
    { "path": "assets/avatars/rank1.jpg", "url": "https://…/storage/v1/object/public/avatars/…" }
  ],
  "generated_at": "2026-08-28T08:00:00Z"
}
```

**34 variables** au total : `period_label`, `period_subtitle`, `participants`, `total_xp`, puis `rankN_name` / `rankN_xp` / `rankN_avatar` pour N de 1 à 10.

`rankN_avatar` est un **chemin local relatif**, jamais une URL : le renderer télécharge les avatars dans le projet avant la capture. Laisser Chromium chercher dix images pendant le rendu casserait le déterminisme.

## Emplacements vides

Un rang sans joueur reçoit les valeurs par défaut du contrat (`Joueur N`, `0`, avatar générique). Le template masque un rang dont `xp = 0`, donc une période à trois participants affiche trois lignes et recentre le podium.

## Opposition à la publication externe

Un joueur dont `profiles.excluded_from_public_media` vaut `TRUE` est **anonymisé ici** :

- `rankN_name` → `Joueur anonyme`
- `rankN_avatar` → avatar générique
- aucune entrée dans `avatars`

Son rang et son XP restent exacts : le retirer fausserait le classement.

⚠️ `build_report_leaderboard` reste **non masquée** : elle sert l'e-mail interne, où nommer les joueurs est légitime. Les deux usages divergent, c'est voulu. La vidéo est le seul chemin qui sort de l'application.

## Erreurs

| Errcode | Cas |
|---------|-----|
| `P0425` | `EMAIL_REPORT_UNKNOWN` : clé de rapport inconnue. |
| `P0425` | `REPORT_VIDEO_UNSUPPORTED` : le rapport n'est pas de type `leaderboard`, seul type doté d'un template vidéo à date. |

## Voir aussi

- [`report_video_renders`](../tables/report_video_renders.md) - état du rendu.
- [`build_report_leaderboard`](./get_email_report_payload.md) - source du classement, non masquée.

## Défis de la période (`new_quests`, migration 110)

Source : `build_report_new_quests` (migration 080). Les défis sont pris dans l'ordre **nouveaux d'abord, puis reconduits**, chaque bloc trié par `display_order` puis `name`. Six emplacements ; au-delà, les défis suivants ne sont pas dans la vidéo, `quest_count` garde le total réel.

**76 variables** : `period_label`, `period_subtitle`, `quest_count`, `new_count`, puis pour N de 1 à 6 :

| Variable | Source | Règle |
|---|---|---|
| `questN_name` | `name` | **Vide = emplacement vide**, le template masque la carte et la ligne. |
| `questN_description` | `description` | Vide → le template affiche l'objectif formaté. |
| `questN_type` / `questN_consumption` | `quest_type` / `consumption_type` | Valeurs d'enum telles quelles ; choisissent le pictogramme et la formulation de l'objectif. |
| `questN_target` | `target_value` | Unité du type (`amount_spent` en centimes, affiché en PdB, jamais en euros). |
| `questN_xp` / `questN_pdb` | `bonus_xp` / `bonus_cashback` | 0 = pas de bonus. |
| `questN_coupon` | `coupon` | Libellé prêt à afficher : « Nom (-10 %) », « Nom (+500 PdB) », ou « Nom ». |
| `questN_badge` | `badge` | Nom du badge ou vide. |
| `questN_max` | `max_completions` | ≥ 1. |
| `questN_scope` | `establishments` | Titres séparés par « , » ; **vide = toutes les tavernes** (aucun scoping, ou scoping couvrant tous les établissements). |
| `questN_new` | bloc `new` / `ongoing` | Booléen. |

Aucune donnée personnelle dans ce payload : pas d'opposition à appliquer, `avatars` est toujours `[]`.

La migration 110 étend aussi `request_report_video_renders` (crons 02:00 / 04:00) et `get_report_video_alerts` (watchdog 05:30) au type `new_quests`, et pose `options.video = true` sur `weekly_new_quests` et `monthly_new_quests`. Ces deux rapports restant inactifs, rien ne se rend ni ne part tant qu'ils ne sont pas activés depuis `/reports`.
