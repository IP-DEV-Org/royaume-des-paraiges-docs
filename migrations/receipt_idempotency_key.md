# Migration: Receipt Idempotency Key

Support de clé d'idempotence pour prévenir les doublons de reçus.

## Informations

| Propriété | Valeur |
|-----------|--------|
| **Date** | 2026-02 |
| **Risque** | Faible (changements additifs uniquement) |
| **Impact** | Aucun downtime requis |

## Objectif

Prévenir la création de reçus en double causée par :
- Double-tap sur le bouton de paiement
- Retry automatique du frontend
- Race conditions réseau

L'application frontend envoie maintenant une clé d'idempotence (`p_idempotency_key`) avec chaque requête de paiement. La base de données et la fonction RPC doivent être mises à jour pour utiliser cette clé pour la déduplication.

---

## 1. Changements de Schéma

### 1.1 Ajout de la colonne idempotency_key

```sql
-- Ajouter la colonne idempotency_key à la table receipts
ALTER TABLE receipts 
ADD COLUMN idempotency_key UUID DEFAULT NULL;

-- Commentaire sur la colonne
COMMENT ON COLUMN receipts.idempotency_key IS 
  'Clé UUID unique envoyée par le client pour éviter les doublons de reçus';
```

### 1.2 Index unique partiel pour l'idempotence

```sql
-- Créer un index unique partiel pour l'idempotence (uniquement sur les valeurs non-null)
CREATE UNIQUE INDEX receipts_idempotency_key_unique 
ON receipts (idempotency_key) 
WHERE idempotency_key IS NOT NULL;
```

> **Note**: L'index partiel permet d'avoir plusieurs `NULL` tout en garantissant l'unicité des clés définies.

### 1.3 Index de sécurité temporelle (Optionnel)

```sql
-- Optionnel: Index pour déduplication basée sur le temps comme sécurité additionnelle
-- Empêche les reçus en double pour le même client/établissement dans les 5 secondes
CREATE UNIQUE INDEX receipts_customer_establishment_time_unique
ON receipts (customer_id, establishment_id)
WHERE created_at > NOW() - INTERVAL '5 seconds';
```

> **Attention**: Cet index optionnel offre une protection supplémentaire mais peut être trop restrictif dans certains cas d'usage légitimes. À évaluer selon les besoins métier.

---

## 2. Mise à Jour de la Fonction RPC

### 2.1 Nouvelle Signature

```sql
CREATE OR REPLACE FUNCTION create_receipt(
  p_customer_id UUID,
  p_establishment_id BIGINT,
  p_payment_methods JSONB,
  p_coupon_ids BIGINT[] DEFAULT ARRAY[]::BIGINT[],
  p_idempotency_key UUID DEFAULT NULL  -- Nouveau paramètre
) RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public'
```

### 2.2 Nouveau Paramètre

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `p_idempotency_key` | `UUID` | ❌ | Clé unique pour éviter les doublons. Si fournie et déjà utilisée, retourne le reçu existant. |

### 2.3 Logique de Déduplication

Ajouter au début de la fonction, après la vérification des permissions :

```sql
-- Étape 0: Vérification d'idempotence
IF p_idempotency_key IS NOT NULL THEN
  SELECT INTO v_existing_receipt
    jsonb_build_object(
      'success', true,
      'idempotent', true,
      'receipt_id', r.id,
      'total_amount', r.amount,
      'message', 'Receipt already exists with this idempotency key'
    )
  FROM receipts r
  WHERE r.idempotency_key = p_idempotency_key;
  
  IF v_existing_receipt IS NOT NULL THEN
    RETURN v_existing_receipt;
  END IF;
END IF;
```

### 2.4 Stockage de la Clé

Modifier l'insertion du receipt (étape 7 actuelle) :

```sql
-- Étape 7: Création du receipt (modifiée)
INSERT INTO receipts (
  customer_id, 
  establishment_id, 
  amount,
  idempotency_key  -- Nouvelle colonne
)
VALUES (
  p_customer_id, 
  p_establishment_id, 
  v_total_amount,
  p_idempotency_key  -- Nouvelle valeur
)
RETURNING id INTO v_receipt_id;
```

### 2.5 Nouvelles Étapes d'Exécution

| # | Étape | Description |
|---|-------|-------------|
| **0** | **Vérification d'idempotence** | Si `p_idempotency_key` fournie, vérifier si un reçu existe déjà |
| 1 | Vérification des permissions | Seuls `employee`, `establishment`, `admin` peuvent créer |
| 2 | Récupération des coefficients | `xp_coefficient` et `cashback_coefficient` du profil |
| ... | ... | (étapes inchangées) |
| 7 | Création du receipt | **Avec stockage de `idempotency_key`** |
| ... | ... | (étapes inchangées) |

### 2.6 Format de Retour Idempotent

Quand un reçu existant est retourné :

```json
{
  "success": true,
  "idempotent": true,
  "receipt_id": 123,
  "total_amount": 2300,
  "message": "Receipt already exists with this idempotency key"
}
```

> **Note**: Le flag `idempotent: true` permet au frontend de distinguer une réponse idempotente d'une nouvelle création.

---

## 3. Exemple d'Utilisation

### TypeScript/JavaScript

```typescript
import { v4 as uuidv4 } from 'uuid';

// Générer une clé d'idempotence unique côté client
const idempotencyKey = uuidv4();

const { data, error } = await supabase.rpc('create_receipt', {
  p_customer_id: customerId,
  p_establishment_id: 1,
  p_payment_methods: [
    { method: 'card', amount: 2500 }
  ],
  p_idempotency_key: idempotencyKey  // Nouvelle option
});

if (data?.success) {
  if (data.idempotent) {
    console.log('Requête dupliquée détectée, reçu existant retourné');
  } else {
    console.log('Nouveau reçu créé:', data.receipt_id);
  }
}
```

### Rétrocompatibilité

```typescript
// Sans clé d'idempotence (backward compatible)
const { data, error } = await supabase.rpc('create_receipt', {
  p_customer_id: customerId,
  p_establishment_id: 1,
  p_payment_methods: [{ method: 'card', amount: 2500 }]
  // p_idempotency_key omis - fonctionne comme avant
});
```

---

## 4. Checklist de Test

### Tests Fonctionnels

- [ ] Vérifier que la migration de schéma s'applique proprement
- [ ] Tester les requêtes dupliquées avec la même clé d'idempotence
  - [ ] La deuxième requête retourne le reçu existant
  - [ ] Le flag `idempotent: true` est présent
  - [ ] Aucun nouveau reçu n'est créé
- [ ] Tester les requêtes uniques avec des clés d'idempotence différentes
  - [ ] Chaque requête crée un nouveau reçu
  - [ ] Le flag `idempotent` est absent ou `false`
- [ ] Tester avec `p_idempotency_key` à `NULL` (rétrocompatibilité)
  - [ ] Les requêtes fonctionnent comme avant la migration
  - [ ] Plusieurs reçus peuvent être créés sans clé
- [ ] Vérifier que le script de rollback fonctionne

### Tests de Charge

- [ ] Simuler des double-taps rapides (< 100ms d'intervalle)
- [ ] Tester les retries automatiques avec la même clé

### Requêtes de Vérification

```sql
-- Vérifier que la colonne existe
SELECT column_name, data_type, is_nullable 
FROM information_schema.columns 
WHERE table_name = 'receipts' AND column_name = 'idempotency_key';

-- Vérifier que l'index existe
SELECT indexname, indexdef 
FROM pg_indexes 
WHERE tablename = 'receipts' AND indexname = 'receipts_idempotency_key_unique';

-- Tester l'unicité (devrait échouer si même clé)
INSERT INTO receipts (customer_id, establishment_id, amount, idempotency_key)
VALUES 
  ('test-uuid', 1, 100, 'same-key-uuid'),
  ('test-uuid', 1, 100, 'same-key-uuid');  -- Devrait lever une erreur
```

---

## 5. Étapes de Déploiement

### Pré-déploiement

1. **Sauvegarde de la base de données**
   ```bash
   # Via Supabase Dashboard ou CLI
   supabase db dump -f backup_before_idempotency.sql
   ```

2. **Vérifier l'état actuel**
   ```sql
   SELECT COUNT(*) FROM receipts;
   SELECT * FROM information_schema.columns WHERE table_name = 'receipts';
   ```

### Déploiement

3. **Appliquer la migration de schéma**
   - Connectez-vous au [Dashboard Supabase](https://app.supabase.com)
   - Sélectionnez le projet `kioysoveqemzjolfwpnu`
   - Allez dans **SQL Editor**
   - Exécutez les scripts de la section 1 (Changements de Schéma)

4. **Mettre à jour la fonction `create_receipt`**
   - Dans **SQL Editor**, exécutez le script complet de la fonction mise à jour
   - Vérifiez que la fonction a été mise à jour :
     ```sql
     SELECT pg_get_functiondef(oid) 
     FROM pg_proc 
     WHERE proname = 'create_receipt';
     ```

5. **Vérification avec paiement de test**
   ```sql
   -- Test avec clé d'idempotence
   SELECT create_receipt(
     p_customer_id := 'test-customer-uuid'::UUID,
     p_establishment_id := 1,
     p_payment_methods := '[{"method": "card", "amount": 100}]'::JSONB,
     p_idempotency_key := 'test-idempotency-key'::UUID
   );
   
   -- Même appel (devrait retourner idempotent: true)
   SELECT create_receipt(
     p_customer_id := 'test-customer-uuid'::UUID,
     p_establishment_id := 1,
     p_payment_methods := '[{"method": "card", "amount": 100}]'::JSONB,
     p_idempotency_key := 'test-idempotency-key'::UUID
   );
   ```

6. **Surveiller les erreurs**
   - Vérifier les logs Supabase pendant 24h
   - Surveiller le taux d'erreurs sur les appels `create_receipt`

---

## 6. Rollback

En cas de problème, exécuter dans l'ordre inverse :

### 6.1 Restaurer la fonction originale

```sql
-- Restaurer la fonction create_receipt sans le paramètre p_idempotency_key
-- (utiliser la version sauvegardée ou recréer depuis la documentation)
```

### 6.2 Supprimer les index

```sql
-- Supprimer l'index de sécurité temporelle (si créé)
DROP INDEX IF EXISTS receipts_customer_establishment_time_unique;

-- Supprimer l'index d'idempotence
DROP INDEX IF EXISTS receipts_idempotency_key_unique;
```

### 6.3 Supprimer la colonne

```sql
-- Supprimer la colonne idempotency_key
ALTER TABLE receipts DROP COLUMN IF EXISTS idempotency_key;
```

### 6.4 Vérification du rollback

```sql
-- Vérifier que la colonne n'existe plus
SELECT column_name FROM information_schema.columns 
WHERE table_name = 'receipts' AND column_name = 'idempotency_key';

-- Vérifier que les index n'existent plus
SELECT indexname FROM pg_indexes 
WHERE tablename = 'receipts' AND indexname LIKE '%idempotency%';
```

---

## 7. Considérations Futures

### Nettoyage des Clés Anciennes

Les clés d'idempotence peuvent être conservées indéfiniment ou nettoyées après un délai raisonnable (ex: 24h). Un job planifié pourrait être ajouté :

```sql
-- Job de nettoyage (optionnel, à planifier)
UPDATE receipts 
SET idempotency_key = NULL 
WHERE idempotency_key IS NOT NULL 
  AND created_at < NOW() - INTERVAL '24 hours';
```

### Métriques

Considérer l'ajout de métriques pour suivre :
- Nombre de requêtes idempotentes détectées
- Taux de doublons évités
- Temps de réponse avec/sans vérification d'idempotence
