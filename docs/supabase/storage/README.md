# Storage - Royaume des Paraiges

## Vue d'ensemble

Le projet utilise **2 buckets** Supabase Storage pour stocker les fichiers.

## Buckets

### avatars

| Propriété | Valeur |
|-----------|--------|
| **ID** | `avatars` |
| **Nom** | `avatars` |
| **Public** | ✅ Oui |
| **Limite de taille** | Aucune |
| **Types MIME autorisés** | Tous |
| **Créé le** | 2025-10-15 |

**URL publique** : `https://kioysoveqemzjolfwpnu.supabase.co/storage/v1/object/public/avatars/`

### content-assets

| Propriété | Valeur |
|-----------|--------|
| **ID** | `content-assets` |
| **Nom** | `content-assets` |
| **Public** | ✅ Oui |
| **Limite de taille** | Aucune |
| **Types MIME autorisés** | Tous |
| **Créé le** | 2026-01-22 |
| **Origine** | Tables de contenu |

**URL publique** : `https://kioysoveqemzjolfwpnu.supabase.co/storage/v1/object/public/content-assets/`

**Structure des dossiers** :

```
content-assets/
├── beers/
│   └── {beer_id}.jpg          # Images des bières
├── establishments/
│   ├── {establishment_id}.jpg  # Images des établissements
│   └── {establishment_id}_logo.jpg  # Logos
└── news/
    └── {news_id}.jpg          # Images des actualités
```

**Note** : Ce bucket contient les images de contenu. Les images sont référencées dans les tables `beers`, `establishments` et `news` via les colonnes `featured_image` et `logo`.

---

## Politiques RLS Storage

### Lecture (SELECT)

| Policy | Bucket | Condition |
|--------|--------|-----------|
| Public Access | avatars | `bucket_id = 'avatars'` |
| Public avatars are viewable by everyone | avatars | `bucket_id = 'avatars'` |
| Public read access for content-assets | content-assets | `bucket_id = 'content-assets'` |

**Résultat** : Tout le monde peut voir les avatars et les images de contenu (bières, établissements, news).

### Upload (INSERT)

| Policy | Condition |
|--------|-----------|
| Users can upload their own avatar | `bucket_id = 'avatars'` ET `name ~ auth.uid() + pattern` |
| Authenticated users can upload avatars | `bucket_id = 'avatars'` |

**Pattern de nommage** :
- `{user_id}.jpg`
- `{user_id}.png`
- `{user_id}-{timestamp}.jpg`

### Mise à jour (UPDATE)

| Policy | Condition |
|--------|-----------|
| Users can update their own avatar | `bucket_id = 'avatars'` ET `name ~ auth.uid() + pattern` |
| Users can update their own avatars | `bucket_id = 'avatars'` ET folder = 'public' |

### Suppression (DELETE)

| Policy | Condition |
|--------|-----------|
| Users can delete their own avatar | `bucket_id = 'avatars'` ET `name ~ auth.uid() + pattern` |
| Users can delete their own avatars | `bucket_id = 'avatars'` |

---

## Convention de Nommage

### Avatars

```
avatars/{user_id}.jpg
avatars/{user_id}.png
avatars/{user_id}-{timestamp}.jpg
```

Exemple :
```
avatars/123e4567-e89b-12d3-a456-426614174000.jpg
avatars/123e4567-e89b-12d3-a456-426614174000-1705678901234.jpg
```

---

## Utilisation dans l'Application

### Upload d'Avatar

```typescript
import { supabase } from '@/src/core/api/supabase';

async function uploadAvatar(userId: string, file: File) {
  const fileExt = file.name.split('.').pop();
  const fileName = `${userId}.${fileExt}`;
  const filePath = fileName;

  // Upload
  const { data, error } = await supabase.storage
    .from('avatars')
    .upload(filePath, file, {
      upsert: true, // Remplace si existe
      contentType: file.type
    });

  if (error) throw error;

  // Récupérer l'URL publique
  const { data: urlData } = supabase.storage
    .from('avatars')
    .getPublicUrl(filePath);

  return urlData.publicUrl;
}
```

### Mise à jour du Profil avec Avatar

```typescript
async function updateProfileAvatar(userId: string, avatarUrl: string) {
  const { error } = await supabase
    .from('profiles')
    .update({ avatar_url: avatarUrl })
    .eq('id', userId);

  if (error) throw error;
}
```

### Suppression d'Avatar

```typescript
async function deleteAvatar(userId: string) {
  // Lister les fichiers de l'utilisateur
  const { data: files } = await supabase.storage
    .from('avatars')
    .list('', {
      search: userId
    });

  if (files && files.length > 0) {
    const filePaths = files.map(f => f.name);
    await supabase.storage.from('avatars').remove(filePaths);
  }

  // Mettre à jour le profil
  await supabase
    .from('profiles')
    .update({ avatar_url: null })
    .eq('id', userId);
}
```

### Affichage d'Avatar

```typescript
// Construire l'URL
const avatarUrl = `https://kioysoveqemzjolfwpnu.supabase.co/storage/v1/object/public/avatars/${userId}.jpg`;

// Ou via Supabase
const { data } = supabase.storage
  .from('avatars')
  .getPublicUrl(`${userId}.jpg`);

// Avec fallback
const displayUrl = profile.avatar_url || '/default-avatar.png';
```

---

## Bonnes Pratiques

### Validation côté client

```typescript
const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp'];

function validateAvatar(file: File): boolean {
  if (file.size > MAX_FILE_SIZE) {
    throw new Error('Fichier trop volumineux (max 5MB)');
  }

  if (!ALLOWED_TYPES.includes(file.type)) {
    throw new Error('Type de fichier non autorisé');
  }

  return true;
}
```

### Optimisation des Images

```typescript
import * as ImageManipulator from 'expo-image-manipulator';

async function resizeAvatar(uri: string): Promise<string> {
  const result = await ImageManipulator.manipulateAsync(
    uri,
    [{ resize: { width: 400, height: 400 } }],
    { compress: 0.8, format: ImageManipulator.SaveFormat.JPEG }
  );

  return result.uri;
}
```

---

## Configuration SQL

### Créer le Bucket

```sql
INSERT INTO storage.buckets (id, name, public)
VALUES ('avatars', 'avatars', true);
```

### Politique de Lecture Publique

```sql
CREATE POLICY "Public Access"
ON storage.objects FOR SELECT
TO public
USING (bucket_id = 'avatars');
```

### Politique d'Upload

```sql
CREATE POLICY "Users can upload their own avatar"
ON storage.objects FOR INSERT
TO authenticated
WITH CHECK (
  bucket_id = 'avatars'
  AND (
    name ~ (auth.uid()::text || '-%')
    OR name = (auth.uid()::text || '.jpg')
    OR name = (auth.uid()::text || '.png')
  )
);
```

---

## Cron Jobs (Non Configuré)

**Note** : `pg_cron` n'est pas activé sur ce projet.

Pour un nettoyage automatique des anciens avatars, vous pourriez utiliser :

```sql
-- Activer pg_cron
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Supprimer les avatars orphelins (sans profil associé)
SELECT cron.schedule(
  'cleanup-orphan-avatars',
  '0 3 * * 0', -- Tous les dimanches à 3h
  $$
    DELETE FROM storage.objects
    WHERE bucket_id = 'avatars'
    AND NOT EXISTS (
      SELECT 1 FROM profiles
      WHERE avatar_url LIKE '%' || objects.name
    )
    AND created_at < NOW() - INTERVAL '7 days'
  $$
);
```

---

## Notes

- Les avatars sont publiquement accessibles (pas d'authentification requise pour les voir)
- Un utilisateur ne peut uploader que ses propres avatars (vérification par UUID)
- `upsert: true` permet de remplacer l'avatar existant
- Pensez à invalider le cache CDN si vous utilisez un CDN externe
