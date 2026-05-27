# Triggers

## Liste des triggers

### Triggers métier (7)

| Trigger | Table | Événement | Fonction | Description |
|---------|-------|-----------|----------|-------------|
| `trigger_create_spending_on_cashback` | `receipt_lines` | AFTER INSERT | `create_spending_from_cashback_payment` | Crée une ligne spending pour chaque paiement cashback |
| `trg_quests_enforce_redundancy` | `quests` | AFTER INSERT/UPDATE | `enforce_quest_redundancy_on_quests` | Bloque les quêtes actives redondantes |
| `trg_qe_enforce_redundancy_insert` | `quests_establishments` | AFTER INSERT | `enforce_quest_redundancy_on_qe_insert` | Vérifie redondance à l'ajout de scope |
| `trg_qe_enforce_redundancy_delete` | `quests_establishments` | AFTER DELETE | `enforce_quest_redundancy_on_qe_delete` | Vérifie redondance à la suppression de scope |
| `trigger_distribute_quest_rewards` | `quest_progress` | AFTER UPDATE | `distribute_quest_rewards` | Distribue les récompenses quand une quête est complétée |
| `gains_refresh_cashback_coefficient` | `gains` | AFTER INSERT/UPDATE/DELETE | `trigger_refresh_cashback_coefficient` | Recalcule le coefficient cashback après modification des gains |
| `trg_identity_photo_cooldown` | `profiles` | BEFORE UPDATE | `enforce_identity_photo_cooldown` | Cooldown 30 jours sur changement de photo d'identité |

### Triggers de validation (2)

| Trigger | Table | Événement | Fonction | Description |
|---------|-------|-----------|----------|-------------|
| `check_deleted_user_on_receipt` | `receipts` | BEFORE INSERT | `create_receipt_deleted_check` | Bloque la création de receipt pour un utilisateur supprimé |
| `check_deleted_user_on_gain` | `gains` | BEFORE INSERT | `check_deleted_user_on_gain` | Bloque la création de gain pour un utilisateur supprimé |

### Triggers auto-timestamp (10)

Triggers `BEFORE UPDATE` qui mettent à jour la colonne `updated_at` automatiquement via `set_updated_at()`.

| Trigger | Table |
|---------|-------|
| `set_beer_styles_updated_at` | `beer_styles` |
| `set_beers_updated_at` | `beers` |
| `set_breweries_updated_at` | `breweries` |
| `set_coupon_templates_updated_at` | `coupon_templates` |
| `set_establishments_updated_at` | `establishments` |
| `set_level_thresholds_updated_at` | `level_thresholds` |
| `set_news_updated_at` | `news` |
| `set_period_reward_configs_updated_at` | `period_reward_configs` |
| `set_profiles_updated_at` | `profiles` |
| `set_quest_progress_updated_at` | `quest_progress` |
| `set_quests_updated_at` | `quests` |
| `set_reward_tiers_updated_at` | `reward_tiers` |

## Details

### trigger_create_spending_on_cashback

- **Table**: `receipt_lines`
- **Fonction**: `create_spending_from_cashback_payment`

```sql
CREATE TRIGGER trigger_create_spending_on_cashback AFTER INSERT ON public.receipt_lines FOR EACH ROW EXECUTE FUNCTION create_spending_from_cashback_payment()
```

### Triggers de prévention des quêtes redondantes (migration 021)

Trois triggers qui s'appuient sur la fonction core `check_quest_redundancy(p_quest_id BIGINT)` pour détecter les redondances de signature fonctionnelle entre quêtes actives.

**Signature fonctionnelle** : `(quest_type, period_type, consumption_type)` avec `NULL IS NOT DISTINCT FROM NULL`.

**Cas de redondance** (entre quêtes actives de même signature) :

| Cas | `conflict_kind` retourné |
|---|---|
| Deux quêtes globales (aucune entrée `quests_establishments`) | `both_global` |
| Une globale + une locale (ou inverse) | `global_vs_local` |
| Deux locales avec ≥ 1 établissement en commun | `locals_overlap` |

**Error code** : `P0421` — custom PostgreSQL errcode (plage `P0xxx` = application-defined).

**Format DETAIL JSON** (pour parsing UX admin en PR#3) :

```json
{
  "conflict_quest_id": 32,
  "conflict_quest_name": "Fidèle parmi les Fidèles",
  "conflict_kind": "both_global",
  "new_quest_id": 41,
  "new_quest_name": "TEST-T1",
  "signature": {
    "quest_type": "quest_completed",
    "period_type": "yearly",
    "consumption_type": null
  }
}
```

**Sémantique** :
- Désactiver une quête (`is_active = false`) est toujours autorisé — ne peut pas créer de conflit.
- Modifier la signature d'une quête inactive est toujours autorisé.
- Ajouter/retirer un lien `quests_establishments` d'une quête **inactive** est toujours autorisé.
- Retirer le **dernier** lien d'une quête locale active la fait basculer en globale — le trigger `AFTER DELETE` détecte ce cas et bloque si conflit.

## Triggers supprimés

### trigger_quest_progress_on_receipt (supprimé le 2026-01-23)

- **Raison** : Ce trigger s'exécutait lors de l'insertion du receipt, mais **avant** que les gains soient créés. Cela causait un bug où les quêtes de type `xp_earned` avaient toujours une progression de 0.
- **Solution** : La mise à jour de la progression des quêtes est maintenant appelée explicitement dans la fonction `create_receipt()` après l'insertion des gains.

