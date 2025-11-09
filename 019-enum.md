# 📝 Tipe Data Enum di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang tipe data Enum di PostgreSQL, yang merupakan tipe data unik yang menggabungkan **human readability** (terlihat seperti string untuk manusia) dengan **compact storage** (disimpan sebagai integer di database). Enum sangat cocok untuk data dengan pilihan tetap dan terbatas seperti status, mood, atau ukuran. Video juga membahas cara menambah/menghapus nilai enum, urutan sorting yang unik, serta perbandingan dengan alternatif lain seperti check constraints.

## 2. Konsep Utama

### a. Karakteristik Dasar Enum

**Dual nature dari Enum:**

- **Untuk manusia**: Terlihat seperti string (readable)
- **Untuk database**: Disimpan sebagai integer (compact)
- **Keuntungan**: Menggabungkan efisiensi storage dengan kemudahan membaca

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

2. **Pilihan terbatas dan finite:**
   ```sql
   -- Contoh: Ukuran pakaian
   CREATE TYPE shirt_size AS ENUM (
       'extra_small',
       'small',
       'medium',
       'large',
       'extra_large'
   );
   ```

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

**Visualisasi urutan:**

```
Awal:      happy → sad → neutral
Add end:   happy → sad → neutral → excited
Add before: happy → afraid → sad → neutral → excited
Add after: happy → afraid → melancholic → sad → neutral → excited
```

### e. Menghapus/Mengubah Nilai Enum (Kompleks!)

**Masalah:** PostgreSQL **tidak mendukung** penghapusan nilai enum secara langsung.

**Solusi:** Drop dan recreate dengan transaction:

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

**Hasil query menunjukkan:**

- `enumtypid`: ID tipe enum
- `enumlabel`: Label yang terlihat ('happy', 'sad', dll)
- `enumsortorder`: Nilai float untuk sorting (1, 1.5, 1.75, 2, dst)

**Bagaimana PostgreSQL handle insert before/after:**

```
Awal:     happy(1.0) → sad(2.0) → neutral(3.0)
+ before: happy(1.0) → afraid(1.5) → sad(2.0) → neutral(3.0)
+ after:  happy(1.0) → afraid(1.5) → melancholic(1.75) → sad(2.0) → neutral(3.0)
```

PostgreSQL menggunakan teknik **"split the difference"** - membagi nilai di tengah-tengah!

**Fungsi utility untuk enum:**

```sql
-- Melihat semua nilai enum dalam urutan yang benar
SELECT enum_range(NULL::mood);
-- Hasil: {happy,afraid,melancholic,sad,neutral,excited}

-- Melihat range dari nilai tertentu
SELECT enum_range('afraid'::mood, 'sad'::mood);
-- Hasil: {afraid,melancholic,sad}
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

**Kerugian:**

- ❌ Storage lebih besar (menyimpan full string, bukan integer)
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

- ✅ Compact storage seperti enum
- ✅ Mudah di-maintain

**Kerugian:**

- ❌ **Completely opaque** - tidak readable tanpa dokumentasi
- ❌ Coworker akan bingung: "Apa arti 1, 2, 3?"

**Perbandingan Storage:**

| Tipe             | Storage           | Readability | Ease of Update |
| ---------------- | ----------------- | ----------- | -------------- |
| **Enum**         | Integer (compact) | ⭐⭐⭐⭐⭐  | ⭐⭐           |
| **Text + Check** | String (larger)   | ⭐⭐⭐⭐⭐  | ⭐⭐⭐⭐⭐     |
| **Integer**      | Integer (compact) | ⭐          | ⭐⭐⭐⭐       |

## 3. Hubungan Antar Konsep

**Alur pemahaman Enum:**

```
1. Deklarasi Type
   ↓
2. Nilai disimpan sebagai integer internally
   ↓
3. Ditampilkan sebagai string untuk human readability
   ↓
4. Sort order berdasarkan urutan deklarasi (bukan alfabetis!)
   ↓
5. Validasi otomatis saat insert/update
   ↓
6. Dapat ditambah nilai baru (di akhir/before/after)
   ↓
7. Tidak bisa langsung hapus nilai (harus recreate)
```

**Trade-offs antara pilihan:**

```
Fixed values + Compact storage needed → Enum ⭐
Frequently changing values → Text + Check Constraint ⭐
Need additional data per value → Lookup Table ⭐
Application-only usage → Integer (acceptable)
```

## 4. Catatan Tambahan / Insight

### 💡 Best Practices

1. **Deklarasi dalam urutan yang meaningful:**

   ```sql
   -- ✅ BAIK: Urutan logis
   CREATE TYPE priority AS ENUM ('low', 'medium', 'high', 'critical');

   -- ❌ BURUK: Urutan random
   CREATE TYPE priority AS ENUM ('high', 'low', 'critical', 'medium');
   ```

2. **Gunakan naming convention yang jelas:**

   ```sql
   -- Untuk type name, gunakan singular noun
   CREATE TYPE status AS ENUM (...);  -- BAIK
   CREATE TYPE statuses AS ENUM (...); -- Kurang baik
   ```

3. **Document urutan enum Anda:**
   ```sql
   -- Buat comment pada type
   COMMENT ON TYPE mood IS 'User mood states, ordered from most positive to least positive';
   ```

### ⚠️ Gotchas dan Pitfalls

1. **Sort order yang unexpected:**

   ```sql
   -- Jika tidak aware, bisa shock saat ORDER BY
   SELECT * FROM table ORDER BY status;
   -- Tidak akan alphabetical!
   ```

2. **Enum removal yang ribet:**

   - Tidak ada `ALTER TYPE ... DROP VALUE`
   - Harus melalui proses panjang: create new → migrate data → drop old

3. **Transaction wajib saat migrate:**

   ```sql
   -- ❌ BERBAHAYA tanpa transaction
   -- Bisa ada inconsistent state jika error di tengah jalan

   -- ✅ AMAN dengan transaction
   BEGIN;
   -- semua operasi di sini
   COMMIT;
   ```

4. **Enum sort order tersimpan sebagai float:**
   - Internal representation: 1.0, 1.5, 1.75, 2.0...
   - Bisa jadi masalah jika insert terlalu banyak di antara dua nilai
   - (Meskipun dalam praktek jarang jadi masalah karena precision float cukup besar)

### 🎯 Decision Framework

**Gunakan Enum jika:**

- ✅ Nilai fixed (≤ 20 opsi)
- ✅ Jarang berubah (< 1x per bulan)
- ✅ Perlu compact storage
- ✅ Readable in database penting
- ✅ Sort order meaningful

**Gunakan Text + Check Constraint jika:**

- ✅ Nilai sering berubah
- ✅ Storage bukan masalah
- ✅ Perlu flexibility tinggi
- ✅ Team sering modify values

**Gunakan Lookup Table jika:**

- ✅ Perlu data tambahan per value
- ✅ Perlu relational integrity
- ✅ Perlu historical tracking
- ✅ Values bisa bertambah via application

### 🔍 Advanced: Kenapa Float untuk Sort Order?

PostgreSQL menggunakan float untuk `enumsortorder` karena:

- Memungkinkan insert di antara dua nilai tanpa re-numbering semua nilai
- Teknik "split the difference" sangat efisien
- Float64 punya precision cukup untuk jutaan insert di antara

```
Ilustrasi:
1.0 dan 2.0 → insert → 1.5
1.0 dan 1.5 → insert → 1.25
1.0 dan 1.25 → insert → 1.125
... dst (bisa sampai precision limit)
```

## 5. Kesimpulan

Enum di PostgreSQL adalah **tipe data hybrid** yang menggabungkan:

- ✅ **Compact storage** (disimpan sebagai integer)
- ✅ **Human readability** (ditampilkan sebagai string)
- ✅ **Data validation** (hanya nilai yang dideklarasikan)
- ✅ **Meaningful sort order** (berdasarkan urutan deklarasi)

**Kapan menggunakan:**

- Pilihan **tetap dan terbatas** (status, kategori, ukuran)
- Perlu **readable representation** di database
- Jarang memerlukan **perubahan nilai**

**Trade-offs:**

- ⚠️ Sulit untuk **remove values** (perlu recreate)
- ⚠️ Sort order **tidak alfabetis** (bisa jadi gotcha)
- ⚠️ Kurang fleksibel dibanding text + check constraint

**Alternatif yang viable:**

- **Text + Check Constraint**: Lebih fleksibel, storage lebih besar
- **Domain**: Reusable, mudah maintain
- **Lookup Table**: Paling fleksibel, untuk complex cases
- **Plain Integer**: Compact tapi tidak readable

**Prinsip akhir:** Pilih enum jika list Anda fixed dan meaningful order penting. Untuk frequent changes, gunakan check constraints atau lookup table.
