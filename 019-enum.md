# 📝 Tipe Data Enum di PostgreSQL (VERSI TERKOREKSI)

## 1. Ringkasan Singkat

Video ini membahas tentang tipe data Enum di PostgreSQL, yang merupakan tipe data unik yang menggabungkan **human readability** (terlihat seperti string untuk manusia) dengan **fixed-size storage** (disimpan dengan ukuran tetap 4 bytes di database). Enum sangat cocok untuk data dengan pilihan tetap dan terbatas seperti status, mood, atau ukuran. Video juga membahas cara menambah/menghapus nilai enum, urutan sorting yang unik, serta perbandingan dengan alternatif lain seperti check constraints.

## 2. Konsep Utama

### a. Karakteristik Dasar Enum

**Dual nature dari Enum:**

- **Untuk manusia**: Terlihat seperti string (readable)
- **Untuk database**: Disimpan dengan **4 bytes fixed size** (bukan "integer" generic)
- **Representasi internal**: Menggunakan **OID (Object Identifier)** dari row di `pg_enum` catalog

  > While OIDs are implemented as unsigned four-byte integers, it's important to understand that the direct integer value itself doesn't represent the ordinal position within the ENUM.

**⚠️ KOREKSI PENTING - Storage Reality:**

```
Enum:  4 bytes + possible alignment padding
Text:  1 byte overhead + actual string length

Contoh perbandingan:
'CA'   → Text: 3 bytes | Enum: 4 bytes  (Text LEBIH KECIL)
'draft' → Text: 6 bytes | Enum: 4 bytes  (Enum LEBIH KECIL)
'published' → Text: 10 bytes | Enum: 4 bytes (Enum LEBIH KECIL)
```

**Kesimpulan Storage:** Enum lebih efisien untuk string ≥ 4 karakter, kurang efisien untuk string pendek.

**Definisi Enum:**

- Enum adalah **finite fixed list** (daftar tetap dan terbatas)
- Anda harus mendeklarasikan semua nilai yang diizinkan di awal
- Memberikan **validasi data otomatis** seperti check constraint

**Contoh deklarasi:**

```sql
-- Membuat tipe enum baru
CREATE TYPE mood AS ENUM (
    'happy',
    'sad',
    'neutral'
);

-- Membuat tabel yang menggunakan enum
CREATE TABLE enum_example (
    current_mood mood
);

-- Insert nilai yang valid
INSERT INTO enum_example (current_mood) VALUES
    ('happy'),
    ('sad'),
    ('neutral');
```

### b. Kapan Menggunakan dan Tidak Menggunakan Enum

**✅ Gunakan Enum ketika:**

1. **Pilihan tetap dan jarang berubah:**

   ```sql
   -- Contoh: Status blog post
   CREATE TYPE post_status AS ENUM (
       'draft',
       'published',
       'archived',
       'destroyed'
   );
   ```

2. **Pilihan terbatas dan finite dengan nilai string panjang:**

   ```sql
   -- Contoh: Ukuran pakaian (lebih dari 3 karakter)
   CREATE TYPE shirt_size AS ENUM (
       'extra_small',
       'small',
       'medium',
       'large',
       'extra_large'
   );
   ```

3. **Perlu sort order yang meaningful (bukan alfabetis)**

**❌ JANGAN gunakan Enum ketika:**

1. **Pilihan sering berubah:**

   - Jika business logic memerlukan penambahan/penghapusan nilai secara terus-menerus
   - Update enum cukup rumit (harus drop dan recreate untuk menghapus nilai)

2. **Butuh lookup table dengan data tambahan:**

   ```sql
   -- ❌ BURUK: Menggunakan enum untuk states
   CREATE TYPE us_state AS ENUM ('CA', 'NY', 'TX', ...);

   -- ✅ BAIK: Gunakan tabel terpisah jika butuh info tambahan
   CREATE TABLE states (
       code VARCHAR(2) PRIMARY KEY,
       name VARCHAR(100),
       capital VARCHAR(100),
       population INTEGER
   );
   ```

3. **Nilai string sangat pendek (≤ 3 karakter):**

   ```sql
   -- ❌ BURUK untuk storage efficiency:
   CREATE TYPE state_code AS ENUM ('CA', 'NY', 'TX');  -- 4 bytes each

   -- ✅ LEBIH EFISIEN:
   -- TEXT atau VARCHAR akan lebih kecil untuk string pendek
   state_code TEXT CHECK (state_code IN ('CA', 'NY', 'TX'))  -- 3 bytes each
   ```

### c. Sort Order: Fitur Unik Enum

**Perbedaan dengan text column:**

Enum **TIDAK** sort secara alfabetis, tetapi berdasarkan **urutan deklarasi**:

```sql
-- Deklarasi enum
CREATE TYPE mood AS ENUM ('happy', 'sad', 'neutral');

-- Insert data
INSERT INTO enum_example (current_mood) VALUES
    ('neutral'), ('happy'), ('sad'), ('happy');

-- Query dengan ORDER BY
SELECT * FROM enum_example ORDER BY current_mood;

-- Hasil: happy, happy, sad, neutral
-- BUKAN: happy, happy, neutral, sad (alfabetis)
```

**Keuntungan sort order ini:**

```sql
-- Contoh: Ukuran pakaian
CREATE TYPE size AS ENUM (
    'extra_small',  -- 1
    'small',        -- 2
    'medium',       -- 3
    'large',        -- 4
    'extra_large'   -- 5
);

-- Saat di-sort, akan muncul dalam urutan yang logis:
-- XS → S → M → L → XL
-- Bukan: extra_large → extra_small → large → medium → small (alfabetis)
```

### d. Menambah Nilai ke Enum yang Sudah Ada

**Menambah di akhir:**

```sql
ALTER TYPE mood ADD VALUE 'excited';
-- 'excited' akan ditambahkan di posisi terakhir
```

**Menambah BEFORE nilai tertentu:**

```sql
ALTER TYPE mood ADD VALUE 'afraid' BEFORE 'sad';
-- Urutan baru: happy, afraid, sad, neutral, excited
```

**Menambah AFTER nilai tertentu:**

```sql
ALTER TYPE mood ADD VALUE 'melancholic' AFTER 'afraid';
-- Urutan baru: happy, afraid, melancholic, sad, neutral, excited
```

**⚠️ CATATAN PENTING:**

- Dari PostgreSQL 9.3+, gunakan `IF NOT EXISTS` untuk idempotency:
  ```sql
  ALTER TYPE mood ADD VALUE IF NOT EXISTS 'excited';
  ```

**Visualisasi urutan:**

```
Awal:       happy → sad → neutral
Add end:    happy → sad → neutral → excited
Add before: happy → afraid → sad → neutral → excited
Add after:  happy → afraid → melancholic → sad → neutral → excited
```

### e. Menghapus/Mengubah Nilai Enum (SANGAT KOMPLEKS!)

**⚠️ PERINGATAN KRITIS:**

PostgreSQL **tidak mendukung** penghapusan nilai enum secara langsung, dan mencoba menghapusnya secara manual dari `pg_enum` **SANGAT BERBAHAYA** karena:

- Nilai masih bisa ada di upper index pages meskipun sudah tidak ada di data
- Bisa merusak index dan menyebabkan database corruption

**Solusi Aman:** Drop dan recreate dengan transaction:

```sql
-- Step 1: Buat enum baru dengan nilai yang diinginkan
CREATE TYPE mood_new AS ENUM ('happy', 'sad', 'neutral', 'afraid');

-- Step 2: Wrap dalam transaction untuk keamanan
BEGIN;

    -- Update data yang tidak valid menjadi nilai default
    UPDATE enum_example
    SET current_mood = 'neutral'
    WHERE current_mood NOT IN ('happy', 'sad', 'neutral', 'afraid');

    -- Ubah kolom ke tipe enum baru
    ALTER TABLE enum_example
    ALTER COLUMN current_mood TYPE mood_new
    USING current_mood::TEXT::mood_new;

COMMIT;

-- Step 3: (Opsional) Drop enum lama dan rename yang baru
DROP TYPE mood;
ALTER TYPE mood_new RENAME TO mood;
```

**Mengapa perlu transaction:**

- Memastikan semua perubahan terjadi **atomically** (semua atau tidak sama sekali)
- Mencegah bad data masuk di tengah-tengah proses
- Jika ada error, semua perubahan akan di-rollback

### f. Melihat "Under the Hood" Enum

**Melihat internal representation:**

```sql
-- Melihat semua enum yang terdaftar
SELECT * FROM pg_catalog.pg_enum;
```

**⚠️ KOREKSI PENTING - Bagaimana Enum Disimpan:**

**Representasi Internal:**

- **Nilai enum disimpan sebagai OID** (Object Identifier) dari row di `pg_enum`
- **BUKAN** langsung sebagai integer 1, 2, 3
- OID adalah 4 bytes identifier yang unik

> While OIDs are implemented as unsigned four-byte integers, it's important to understand that the direct integer value itself doesn't represent the ordinal position within the ENUM.

**Hasil query menunjukkan:**

- `enumtypid`: ID tipe enum
- `enumlabel`: Label yang terlihat ('happy', 'sad', dll)
- `enumsortorder`: **Nilai float/numeric untuk sorting** (ini yang float, bukan representasi internal!)

**Bagaimana PostgreSQL handle insert before/after:**

```
Awal:     happy(1.0) → sad(2.0) → neutral(3.0)
+ before: happy(1.0) → afraid(1.5) → sad(2.0) → neutral(3.0)
+ after:  happy(1.0) → afraid(1.5) → melancholic(1.75) → sad(2.0) → neutral(3.0)
```

PostgreSQL menggunakan teknik **"split the difference"** pada `enumsortorder` field (bukan pada representasi internal!).

**Fungsi utility untuk enum:**

```sql
-- Melihat semua nilai enum dalam urutan yang benar
SELECT enum_range(NULL::mood);
-- Hasil: {happy,afraid,melancholic,sad,neutral,excited}

-- Melihat range dari nilai tertentu
SELECT enum_range('afraid'::mood, 'sad'::mood);
-- Hasil: {afraid,melancholic,sad}

-- Fungsi lainnya
SELECT enum_first(NULL::mood);  -- Nilai pertama
SELECT enum_last(NULL::mood);   -- Nilai terakhir
```

### g. Alternatif: Check Constraints vs Enum

**1. Text dengan Check Constraint:**

```sql
CREATE TABLE posts (
    status TEXT CHECK (status IN ('draft', 'published', 'archived'))
);
```

**Keuntungan:**

- ✅ Lebih mudah di-maintain (tinggal ALTER TABLE untuk ubah constraint)
- ✅ Tidak perlu create/drop type
- ✅ Lebih fleksibel untuk perubahan
- ✅ Storage lebih efisien untuk string pendek (< 4 chars)

**Kerugian:**

- ❌ Storage lebih besar untuk string panjang (> 4 chars)
- ❌ Tidak ada sort order khusus

**2. Domain dengan Check Constraint:**

```sql
CREATE DOMAIN post_status AS TEXT
CHECK (VALUE IN ('draft', 'published', 'archived'));

CREATE TABLE posts (
    status post_status
);
```

**Keuntungan:**

- ✅ Reusable (bisa dipakai di banyak tabel)
- ✅ Mudah diubah
- ✅ Centralized validation

**3. Plain Integer:**

```sql
CREATE TABLE posts (
    status INTEGER CHECK (status BETWEEN 1 AND 4)
);
-- 1=draft, 2=published, 3=archived, 4=destroyed
```

**Keuntungan:**

- ✅ Compact storage (4 bytes seperti enum)
- ✅ Mudah di-maintain

**Kerugian:**

- ❌ **Completely opaque** - tidak readable tanpa dokumentasi
- ❌ Coworker akan bingung: "Apa arti 1, 2, 3?"

**Perbandingan Storage (DIKOREKSI):**

| Tipe             | Storage (per value)    | Readability | Ease of Update | Best For                |
| ---------------- | ---------------------- | ----------- | -------------- | ----------------------- |
| **Enum**         | 4 bytes (fixed)        | ⭐⭐⭐⭐⭐  | ⭐⭐           | Strings ≥ 4 chars       |
| **Text + Check** | 1 byte + string length | ⭐⭐⭐⭐⭐  | ⭐⭐⭐⭐⭐     | Any length, flexibility |
| **Integer**      | 4 bytes                | ⭐          | ⭐⭐⭐⭐       | Application layer only  |
| **Domain**       | Same as base type      | ⭐⭐⭐⭐    | ⭐⭐⭐⭐⭐     | Reusable constraints    |

**Contoh Real World:**

```sql
-- Enum LEBIH EFISIEN:
CREATE TYPE long_status AS ENUM ('draft', 'published', 'archived', 'destroyed');
-- 4 bytes per value

-- TEXT LEBIH EFISIEN:
CREATE TABLE states (
    code TEXT CHECK (code IN ('CA', 'NY', 'TX'))  -- 3 bytes per value
);
```

## 3. Hubungan Antar Konsep

**Alur pemahaman Enum:**

```
1. Deklarasi Type
   ↓
2. Nilai disimpan sebagai OID (4 bytes) internally
   ↓
3. OID merujuk ke pg_enum catalog row
   ↓
4. Ditampilkan sebagai string untuk human readability
   ↓
5. Sort order berdasarkan enumsortorder field (bukan OID, bukan alfabetis!)
   ↓
6. Validasi otomatis saat insert/update
   ↓
7. Dapat ditambah nilai baru (di akhir/before/after)
   ↓
8. TIDAK BISA hapus nilai secara aman (bisa corrupt index)
```

**Trade-offs antara pilihan (UPDATED):**

```
Fixed values + Long strings (≥4 chars) + Meaningful order → Enum ⭐
Fixed values + Short strings (<4 chars) → Text + Check ⭐
Frequently changing values → Text + Check Constraint ⭐
Need additional data per value → Lookup Table ⭐
Reusable across tables → Domain ⭐
Application-only usage → Integer (acceptable)
```

## 4. Kesimpulan

Enum di PostgreSQL adalah **tipe data hybrid** yang menggabungkan:

- ✅ **Fixed-size storage** (4 bytes OID)
- ✅ **Human readability** (ditampilkan sebagai string)
- ✅ **Data validation** (hanya nilai yang dideklarasikan)
- ✅ **Meaningful sort order** (berdasarkan enumsortorder, bukan alfabetis)

**Kapan menggunakan:**

- Pilihan **tetap dan terbatas** dengan string ≥ 4 karakter
- Perlu **readable representation** di database
- Jarang memerlukan **perubahan nilai**
- Meaningful order penting

**Trade-offs:**

- ⚠️ **TIDAK BISA** remove values secara aman
- ⚠️ Sort order **tidak alfabetis** (bisa jadi gotcha)
- ⚠️ Kurang fleksibel dibanding text + check constraint
- ⚠️ Tidak selalu lebih compact (4 bytes fixed vs text variable)

**Alternatif yang viable:**

- **Text + Check Constraint**: Lebih fleksibel, lebih baik untuk string pendek
- **Domain**: Reusable, mudah maintain
- **Lookup Table**: Paling fleksibel, untuk complex cases
- **Plain Integer**: Compact tapi tidak readable

**Prinsip akhir:**

- Pilih enum jika list fixed, string panjang (≥4 chars), dan meaningful order penting
- Untuk frequent changes atau string pendek, gunakan check constraints
- Untuk complex requirements, gunakan lookup table
- Selalu pertimbangkan storage efficiency berdasarkan panjang string actual
