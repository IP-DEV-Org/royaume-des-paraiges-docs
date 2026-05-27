# Migrations Supabase - Royaume des Paraiges

Ce dossier contient les scripts de migration SQL pour la base de donnees Supabase.

## Liste des Migrations

| Fichier | Description | Date |
|---------|-------------|------|
| `phase4_data_migration.sql` | Migration systeme de coupons administrable | 2026-01 |
| `fix_quest_progress_timing` | Correction timing mise a jour progression quetes | 2026-01-23 |

## Comment executer une migration

1. Connectez-vous au [Dashboard Supabase](https://app.supabase.com)
2. Selectionnez le projet `kioysoveqemzjolfwpnu`
3. Allez dans **SQL Editor**
4. Copiez-collez le contenu du fichier de migration
5. Executez le script

## Ordre d'execution

Les migrations doivent etre executees dans l'ordre chronologique. Chaque migration verifie les pre-requis avant de s'executer.

## Migration Phase 4 : Systeme de Coupons Administrable

### Description

Cette migration met en place le nouveau systeme de coupons administrable :

1. **Creation des templates de coupons** (5 templates)
   - Coupon Hebdo 50€ (3.90€)
   - Coupon Frequence 5%
   - Recompense Champion (10€)
   - Recompense Podium (5€)
   - Recompense Top 10 (10%)

2. **Creation des reward_tiers** (9 tiers)
   - 3 tiers hebdomadaires (Champion, Podium, Top 10)
   - 3 tiers mensuels (Champion, Podium, Top 10)
   - 3 tiers annuels (Champion, Podium, Top 10)

3. **Migration des coupons existants**
   - Marquage des anciens coupons avec `distribution_type = 'trigger_legacy'`

### Pre-requis

Avant d'executer cette migration, les tables suivantes doivent exister :
- `coupon_templates`
- `reward_tiers`
- `coupons` (avec colonne `distribution_type`)

### Verification

Apres execution, verifier avec :

```sql
-- Templates crees
SELECT * FROM coupon_templates ORDER BY created_at;

-- Tiers avec templates associes
SELECT
  rt.name,
  rt.rank_from,
  rt.rank_to,
  rt.period_type,
  ct.name as template_name,
  ct.amount,
  ct.percentage
FROM reward_tiers rt
LEFT JOIN coupon_templates ct ON rt.coupon_template_id = ct.id
ORDER BY rt.period_type, rt.display_order;

-- Distribution des coupons par type
SELECT distribution_type, COUNT(*) as count
FROM coupons
GROUP BY distribution_type;
```

## Migration fix_quest_progress_timing (2026-01-23)

### Description

Correction d'un bug ou les quetes de type `xp_earned` n'etaient jamais mises a jour.

### Probleme

Le trigger `trigger_quest_progress_on_receipt` s'executait lors de l'insertion du receipt (etape 6 de `create_receipt`), mais les gains XP etaient crees plus tard (etape 10). La fonction `calculate_quest_progress` pour `xp_earned` cherchait les gains qui n'existaient pas encore, donc retournait toujours 0.

### Solution

1. Suppression du trigger `trigger_quest_progress_on_receipt`
2. Ajout d'un appel explicite a `update_quest_progress_for_receipt()` dans `create_receipt()` apres l'insertion des gains

### Changements

```sql
-- Trigger supprime
DROP TRIGGER IF EXISTS trigger_quest_progress_on_receipt ON receipts;

-- Fonction create_receipt modifiee pour appeler update_quest_progress_for_receipt
-- apres l'etape 10 (insertion des gains)
```

### Verification

```sql
-- Verifier que le trigger n'existe plus
SELECT trigger_name FROM information_schema.triggers
WHERE event_object_table = 'receipts';

-- Tester la progression d'une quete xp_earned
SELECT calculate_quest_progress(
  'customer_uuid'::uuid,
  quest_id,
  '2026-01'
);
```

## Bonnes Pratiques

1. **Toujours tester en local** avant d'executer en production
2. **Faire une sauvegarde** avant toute migration majeure
3. **Verifier les resultats** avec les requetes de verification fournies
4. **Documenter les changements** dans ce README

## Rollback

En cas de probleme, chaque migration devrait avoir un script de rollback. Contacter l'administrateur de la base de donnees si necessaire.
