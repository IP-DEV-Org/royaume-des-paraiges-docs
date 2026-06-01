# Design / RFC — Dettes inter-établissements (PdB récompense)

> **Statut : brouillon de conception (non implémenté).** Document de travail à affiner avec Basile avant toute migration. Aucune table ni RPC décrite ici n'existe encore en base.
>
> Séquencement validé (juin 2026) : la page `/analytics` (timeline par journée fiscale) est livrée **sans** ce modèle ; les PdB récompense y sont affichés dans un bloc global provisoire « Royaume — toutes enseignes ». Ce document décrit le chantier qui permettra, à terme, de **ventiler les PdB récompense par établissement**.

---

## 1. Problème

Les Paraiges de Bronze (PdB) **organiques** (`gains.source_type = 'receipt'`) portent un `establishment_id` : on sait où ils ont été générés. Les PdB **récompense** (`bonus_cashback_*`, crédités via `credit_bonus_cashback()`) sont insérés avec `establishment_id = NULL` **et** `receipt_id = NULL` : ils ne sont rattachables à **aucun** établissement.

Or une récompense est presque toujours « financée » par des dépenses réparties sur **plusieurs** établissements. Sans mécanisme d'attribution, un établissement peut investir (générer la dépense qualifiante) sans jamais être compensé, tandis qu'un autre encaisse la dépense des PdB.

### Exemple de référence (Basile)

Quête « **consomme 3 bières → 15 PdB** ». Le client achète :
- 2 bières à l'**établissement A** ;
- 1 bière à l'**établissement B**.

La récompense de **15 PdB** provient donc d'une dépense qualifiante répartie **2/3 chez A, 1/3 chez B**.

Plus tard, le client **dépense ces 15 PdB chez B**. Énoncé par Basile :

> « L'établissement B doit avoir une dette de 2/3 de ces PdB envers l'établissement A, sinon l'établissement A est perdant. »

C'est cette mécanique de **dette inter-établissements** que ce document vise à modéliser.

---

## 2. Concepts & vocabulaire

| Terme | Définition |
|---|---|
| **Dépense qualifiante** | Les receipts / consommations qui ont fait progresser une quête jusqu'à son déclenchement. Réparties sur 1..N établissements. |
| **Attribution** | Répartition d'une récompense (un `gains` bonus) entre établissements, au **prorata** de la dépense qualifiante. Σ(parts) = `gains.cashback_money`. |
| **Établissement source / financeur** | Établissement chez qui une partie de la dépense qualifiante a eu lieu (A et B dans l'exemple). |
| **Établissement de rédemption** | Établissement où le client **dépense** les PdB (`receipt_lines.payment_method = 'cashback'`). |
| **Dette** | Obligation financière entre deux établissements, créée au moment de la rédemption des PdB récompense. |

---

## 3. ⚠️ Question centrale à trancher — sens de la dette

L'exemple de Basile implique : **B (rédemption) doit à A (source)**.

Selon le modèle comptable retenu, le sens « naturel » peut différer. Deux lectures cohérentes coexistent :

### Lecture 1 — Remboursement de coût (« cost-reimbursement »)
À la rédemption, B offre une valeur de 15 PdB au client (remise / produit offert) : **B avance le coût**. Si la récompense était « due » à 2/3 par A, alors **A doit rembourser B** de 2/3 → *A doit à B*.

### Lecture 2 — Attribution marketing (lecture de Basile)
A a « investi » pour fidéliser le client (sa dépense a généré la récompense). Quand le client revient dépenser chez B, **B capte une vente grâce à l'effort de A** → *B doit à A*.

➡️ **À confirmer avec Basile** : c'est la **lecture 2** (B → A) qui est énoncée dans l'exemple. Il faut valider que c'est bien l'intention, car elle implique que l'établissement de rédemption (B) supporte **à la fois** le coût de la remise PdB **et** une dette envers les financeurs. Le reste du document est écrit de façon **agnostique au sens** : on stocke `debtor` / `creditor` explicitement, le sens se règle dans la logique de génération de dette (§6).

---

## 4. Pré-requis : tracer la dépense qualifiante

Aujourd'hui `credit_bonus_cashback()` ne reçoit ni les receipts ni les établissements à l'origine de la récompense. Il faut d'abord **capter l'attribution** au moment du crédit.

Sources de l'attribution selon `source_type` :

| `source_type` | Base d'attribution proposée | Remarque |
|---|---|---|
| `bonus_cashback_quest` | Receipts ayant fait progresser la quête sur la période (`quest_progress` / `quest_completion_logs` → receipts → `establishment_id`). | **Cœur du sujet.** La clé de prorata dépend du `quest_type` (cf. §4.1). |
| `bonus_cashback_leaderboard` | Dépense totale du client sur la période par établissement. | À débattre : un classement récompense l'activité globale, pas une quête précise. |
| `bonus_cashback_trigger` | Receipt déclencheur (un seul établissement, en général). | Attribution triviale (100 % au receipt). |
| `bonus_cashback_manual` | Aucune base métier. | Reste **central / non attribué** (cf. §7). |
| `bonus_cashback_migration` | Historique. | Reste central / non attribué. |
| `rollback_beta_correction` | Correction (montant négatif). | Doit **annuler** l'attribution correspondante au prorata. |

### 4.1 Clé de prorata selon le type de quête

La « dépense qualifiante » n'est pas toujours un montant en euros :

- `amount_spent` / `cashback_earned` → prorata au **montant** dépensé par établissement.
- `consumption_count` (ex. 3 bières) → prorata au **nombre d'items qualifiants** par établissement (2 vs 1 → 2/3, 1/3).
- `orders_count` → prorata au **nombre de tickets** par établissement.
- `establishments_visited` → ? (1 visite = 1 établissement) — répartition égale ? **à définir**.
- `quest_completed` (méta-quête) → récursif : attribution = somme pondérée des attributions des sous-quêtes complétées. **à définir**.

➡️ **Question ouverte** : valider la clé de prorata par `quest_type`, et la fenêtre de dépense prise en compte (toute la période vs uniquement les receipts entre l'avant-dernière et la dernière progression).

---

## 5. Modèle de données proposé

### 5.1 `pdb_reward_attributions` — d'où vient une récompense

Une ligne par (récompense, établissement source).

| Colonne | Type | Description |
|---|---|---|
| `id` | bigint PK | |
| `gain_id` | bigint FK `gains(id)` ON DELETE CASCADE | La récompense attribuée. |
| `establishment_id` | int FK `establishments(id)` | Établissement source (financeur). |
| `share_cents` | int | Part de PdB attribuée à cet établissement. **Σ par `gain_id` = `gains.cashback_money`**. |
| `weight_basis` | text | Clé de prorata utilisée (`amount` / `items` / `orders` / …) — audit. |
| `created_at` | timestamptz default now() | |

Contraintes : `UNIQUE(gain_id, establishment_id)`. Invariant applicatif : `SUM(share_cents) WHERE gain_id = X` = `cashback_money` du gain X (à vérifier par test, voir §9).

### 5.2 `establishment_debts` — dettes générées à la rédemption

Une ligne par dette créée lorsqu'un client dépense des PdB récompense.

| Colonne | Type | Description |
|---|---|---|
| `id` | bigint PK | |
| `debtor_establishment_id` | int FK `establishments(id)` | Qui doit. |
| `creditor_establishment_id` | int FK `establishments(id)` | Qui est dû. |
| `amount_cents` | int | Montant de la dette (PdB). |
| `customer_id` | uuid FK `profiles(id)` | Client à l'origine (traçabilité). |
| `redemption_receipt_id` | bigint FK `receipts(id)` ON DELETE … | Receipt de rédemption. |
| `source_gain_id` | bigint FK `gains(id)` | Récompense d'origine (traçabilité). |
| `status` | text CHECK (`open` / `settled` / `void`) | Cycle de vie. |
| `created_at` / `settled_at` | timestamptz | |

Index : `(creditor_establishment_id, status)`, `(debtor_establishment_id, status)`.
Contrainte : `CHECK (debtor_establishment_id <> creditor_establishment_id)` (pas de dette envers soi-même — la part « rédemption = source » s'auto-annule).

### 5.3 (Optionnel) `establishment_debt_settlements` — soldes / netting

Pour matérialiser les règlements périodiques (ex. mensuels) : netting des dettes ouvertes entre paires d'établissements → une transaction nette. À concevoir en phase 2.

### 5.4 Vue d'agrégation

`establishment_debt_balances` : par établissement, total dû / total créancier / solde net, filtrable par période. Alimente une page admin `/reconciliation/debts` (ou un onglet analytics).

---

## 6. Logique (flux)

### 6.1 À la génération d'une récompense (earning)
1. `credit_bonus_cashback()` (ou un wrapper) calcule l'attribution par établissement selon §4.
2. Insère les lignes `pdb_reward_attributions` (Σ = montant du gain).
3. Si aucune base d'attribution (manuel/migration) → pas d'attribution (PdB « central », cf. §7).

### 6.2 À la rédemption (spending)
Quand un client paie avec des PdB (`receipt_lines.payment_method = 'cashback'`) chez l'établissement R :
1. Déterminer **quels PdB** sont consommés (organiques vs récompense, et de quelles récompenses) → **politique de consommation à définir** (§8).
2. Pour la fraction de PdB **récompense** consommée, lire les `pdb_reward_attributions` correspondantes.
3. Pour chaque établissement source S ≠ R : créer une `establishment_debts` (sens à fixer §3) d'un montant = part consommée attribuée à S.
4. La part attribuée à R lui-même ne génère pas de dette (auto-annulation).

### 6.3 Aux corrections / annulations
- Suppression d'un receipt de rédemption (`admin_delete_receipt`) → `void` / suppression des dettes liées (`redemption_receipt_id`).
- `rollback_beta_correction` sur une récompense → annuler/réduire les attributions, et les dettes déjà générées si la récompense est reprise.

---

## 7. PdB « centraux » (non attribuables)

Les bonus `manual` / `migration` (et potentiellement une part des leaderboard) n'ont pas de base d'attribution établissement. Options :
- **(a)** Les laisser non attribués → à la rédemption, aucune dette (le Royaume / la holding absorbe). Ligne « central » dans les soldes.
- **(b)** Les attribuer à une entité « Royaume » fictive créancière/débitrice.

➡️ **Question ouverte** : qui supporte le coût des PdB sans origine établissement ?

---

## 8. Politique de consommation des PdB (FIFO ?)

Un solde client est un mélange de PdB organiques et récompense, de sources diverses. À la rédemption d'un montant partiel, **quels PdB sont dépensés en premier ?**
- **FIFO** par date de génération (`gains.created_at`).
- **Organiques d'abord** (les récompense, plus « coûteuses » en dettes, restent au solde).
- **Récompense d'abord**.
- **Au prorata** du solde.

Ce choix change fortement le volume de dettes générées. ➡️ **À trancher avec Basile.** Implique probablement de tenir un **registre de lots** (lot = un gain avec son reliquat consommable) plutôt qu'un simple solde agrégé — gros impact sur `gains` / `spendings`.

---

## 9. Intégrations & impacts

- **Backend crédit bonus** : `credit_bonus_cashback()` + les fonctions de quêtes / leaderboard (cf. migrations archive 040) doivent appeler la logique d'attribution.
- **`create_receipt`** (Scanner / Waiters) : chemin de **dépense** des PdB → doit déclencher la génération de dettes. Impacte donc aussi ces deux apps (backend partagé).
- **Analytics** : une fois `pdb_reward_attributions` peuplé, `get_analytics_timeline` pourra **ventiler les PdB récompense par établissement** (et le bloc global provisoire de `/analytics` disparaît). Voir [`functions/get_analytics_timeline.md`](../functions/get_analytics_timeline.md).
- **Page admin dédiée** : visualisation des soldes de dettes + règlement (`/reconciliation/debts` ou section analytics).
- **Garde-fou solde négatif** (`trg_enforce_non_negative_cashback`, migration 043) : à articuler avec un éventuel registre de lots.

---

## 10. Questions ouvertes (récapitulatif)

1. **Sens de la dette** : confirmer lecture 2 (B → A, l'établissement de rédemption doit aux financeurs) vs lecture 1 (remboursement de coût A → B). *(§3)*
2. **Clé de prorata** par `quest_type` et fenêtre de dépense qualifiante retenue. *(§4.1)*
3. **Attribution des leaderboard** : sur quelle base ? *(§4)*
4. **PdB centraux** (manuel/migration) : qui absorbe ? *(§7)*
5. **Politique de consommation** des PdB à la rédemption (FIFO, organiques d'abord, lots…). *(§8)*
6. **Règlement** : netting périodique automatique ou suivi manuel ? *(§5.3)*
7. **Rétroactif** : applique-t-on le modèle à l'historique (attribution rétro des récompenses passées) ou seulement au flux futur ?

---

## 11. Esquisse de plan de mise en œuvre

1. **Phase 0 — Décisions** : trancher les 7 questions ci-dessus (atelier avec Basile).
2. **Phase 1 — Attribution (earning)** : migration `pdb_reward_attributions` + branchement dans `credit_bonus_cashback` et les quêtes ; tests d'invariant (Σ parts = montant gain). Backfill optionnel.
3. **Phase 2 — Dettes (spending)** : migration `establishment_debts` + génération à la rédemption dans `create_receipt` (impacte Scanner/Waiters) ; politique de consommation.
4. **Phase 3 — Restitution** : vue de soldes + page admin + ventilation des PdB récompense dans `get_analytics_timeline` (suppression du bloc global provisoire de `/analytics`).
5. **Phase 4 — Règlement** : netting / settlements (si retenu).

---

*Document créé en juin 2026 dans le prolongement de la refonte `/analytics`. À mettre à jour au fil des décisions.*
