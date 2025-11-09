# 📝 Tipe Data Boolean di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang tipe data Boolean di PostgreSQL. Meskipun terlihat sederhana (hanya true/false), ada beberapa hal menarik tentang implementasi Boolean di Postgres. Berbeda dengan database lain yang menggunakan integer kecil sebagai alias, PostgreSQL memiliki tipe data Boolean yang sesungguhnya dengan ukuran hanya 1 byte dan dapat menerima berbagai format input yang akan otomatis dikonversi.

## 2. Konsep Utama

### a. Boolean sebagai Tipe Data Native di PostgreSQL

- PostgreSQL memiliki **tipe data Boolean yang sesungguhnya**, bukan hanya alias untuk integer seperti di database lain.
- Beberapa database lain menggunakan "tiny integer" di balik layar untuk memenuhi standar SQL, tetapi Postgres memiliki tipe data khusus untuk Boolean.
- Sesuai filosofi Postgres: "ada tipe data untuk segalanya", termasuk Boolean.

**Contoh deklarasi:**

```sql
CREATE TABLE boolean_example (
    status BOOLEAN
);
```

### b. Tiga Kemungkinan Nilai Boolean

Boolean di PostgreSQL dapat memiliki tiga nilai:

- **`TRUE`** - nilai benar
- **`FALSE`** - nilai salah
- **`NULL`** - mewakili **state yang tidak diketahui** (unknown state)

Ini memberikan fleksibilitas untuk kasus di mana status belum ditentukan atau tidak diketahui.

### c. Fleksibilitas Input: Berbagai Format yang Diterima

PostgreSQL sangat fleksibel dalam menerima input untuk tipe Boolean. Berikut format-format yang dapat diterima:

| Format Input                    | Hasil Konversi |
| ------------------------------- | -------------- |
| `true`, `'true'`, `'t'`         | `TRUE`         |
| `false`, `'false'`, `'f'`       | `FALSE`        |
| `'1'` (string), `'on'`, `'yes'` | `TRUE`         |
| `'0'` (string), `'off'`, `'no'` | `FALSE`        |
| `NULL`                          | `NULL`         |

**Contoh insert dengan berbagai format:**

```sql
INSERT INTO boolean_example (status) VALUES
    (true),           -- boolean literal
    (false),          -- boolean literal
    ('t'),            -- string 't'
    ('true'),         -- string 'true'
    ('f'),            -- string 'f'
    ('false'),        -- string 'false'
    ('1'),            -- string '1'
    ('0'),            -- string '0'
    ('on'),           -- string 'on'
    ('off'),          -- string 'off'
    ('yes'),          -- string 'yes'
    ('no'),           -- string 'no'
    (NULL);           -- null value
```

**Hasil query:**

```sql
SELECT * FROM boolean_example;
```

Semua nilai di atas akan dikonversi dan ditampilkan sebagai `t` (true), `f` (false), atau `null`.

### d. Casting Eksplisit vs Implicit

Ada perbedaan penting antara **explicit cast** dan **insert langsung**:

**✅ Explicit cast - BERFUNGSI:**

```sql
SELECT 1::BOOLEAN;        -- menggunakan integer literal
SELECT '1'::BOOLEAN;      -- menggunakan string
```

**❌ Insert langsung integer - TIDAK BERFUNGSI:**

```sql
-- Ini akan ERROR:
INSERT INTO boolean_example (status) VALUES (1);
```

**Poin penting:**

- Saat melakukan **explicit cast**, PostgreSQL dapat mengkonversi integer `1` atau `0` menjadi Boolean
- Saat melakukan **insert**, harus menggunakan string `'1'` atau `'0'`, bukan integer langsung

### e. Ukuran Storage: Hanya 1 Byte!

Boolean di PostgreSQL sangat efisien dalam penggunaan storage:

```sql
SELECT pg_column_size(true::BOOLEAN);   -- Hasil: 1 byte
SELECT pg_column_size(1::INTEGER);      -- Hasil: 4 bytes
SELECT pg_column_size(1::SMALLINT);     -- Hasil: 2 bytes (alias: int2)
```

**Perbandingan ukuran:**

- `BOOLEAN`: **1 byte**
- `SMALLINT` (int2): **2 bytes** (2x lebih besar)
- `INTEGER`: **4 bytes** (4x lebih besar)

**Mengapa ini penting:**

- Meskipun `SMALLINT` memiliki range hingga ±32,000, kita hanya butuh 0 atau 1
- Menggunakan Boolean menghemat 50% storage dibanding `SMALLINT`
- Gunakan tipe data yang **paling tepat merepresentasikan data Anda**

### f. Konversi ke Tipe Data Lain

Boolean dapat dikonversi ke integer dan tipe lainnya:

```sql
-- Konversi Boolean ke integer
SELECT true::INTEGER;     -- Hasil: 1
SELECT false::INTEGER;    -- Hasil: 0

-- Memeriksa tipe data
SELECT pg_typeof(true::BOOLEAN);   -- Hasil: 'boolean'
SELECT pg_typeof(true::INTEGER);   -- Hasil: 'integer'
```

## 3. Hubungan Antar Konsep

**Alur pemahaman:**

1. **Tipe Data Native** → PostgreSQL memiliki implementasi Boolean yang sesungguhnya, bukan hanya wrapper integer
2. **Fleksibilitas Input** → Berbagai format input (true/false, 1/0, yes/no, on/off) membuat interaksi lebih mudah
3. **Automatic Coercion** → Semua format input otomatis dikonversi ke representasi Boolean internal
4. **Efisiensi Storage** → Ukuran 1 byte membuat Boolean sangat efisien untuk menyimpan data ya/tidak
5. **Type Safety** → Meskipun fleksibel, PostgreSQL tetap menjaga keamanan tipe (tidak bisa insert integer langsung tanpa cast)

**Filosofi design:**
PostgreSQL mengutamakan **ketepatan tipe data** untuk merepresentasikan domain data. Boolean untuk true/false, bukan integer yang di-abuse.

## 4. Catatan Tambahan / Insight

### 💡 Tips Penggunaan

1. **Selalu gunakan Boolean untuk data true/false:**

   - Lebih semantik dan jelas maksudnya
   - Lebih efisien (1 byte vs 2-4 bytes)
   - Type-safe (tidak bisa secara tidak sengaja memasukkan nilai di luar 0/1)

2. **Input paling jelas: gunakan `true`/`false` literal:**

   ```sql
   -- ✅ BAGUS: Paling jelas dan eksplisit
   INSERT INTO users (is_active) VALUES (true);

   -- ⚠️ KURANG JELAS: Meskipun valid
   INSERT INTO users (is_active) VALUES ('1');
   ```

3. **NULL untuk unknown state:**
   - Gunakan NULL jika status belum diketahui
   - Contoh: `is_verified` bisa NULL jika verifikasi belum dilakukan

### ⚠️ Kapan TIDAK Menggunakan Boolean

**Jangan gunakan Boolean jika:**

1. **Lebih dari dua state (plus unknown):**

   ```sql
   -- ❌ BURUK: Menggunakan multiple boolean
   status_pending BOOLEAN,
   status_processing BOOLEAN,
   status_completed BOOLEAN

   -- ✅ BAIK: Gunakan ENUM atau string
   status VARCHAR(20) CHECK (status IN ('pending', 'processing', 'completed'))
   ```

2. **Untuk operasi bitwise:**

   - Jika Anda perlu melakukan operasi bit (AND, OR, XOR), gunakan integer types
   - Boolean tidak dirancang untuk bitwise operations

3. **Untuk flag bitmask:**
   ```sql
   -- Jika Anda butuh menyimpan multiple flags dalam satu kolom
   -- Gunakan INTEGER dengan bitwise operations
   permissions INTEGER; -- bit 1: read, bit 2: write, bit 3: delete
   ```

### 🎯 Best Practices

- **Nama kolom yang semantik:** `is_active`, `has_permission`, `can_edit` lebih jelas daripada `status`, `flag`
- **Konsisten dengan format input:** Jika team Anda menggunakan `true`/`false`, stick dengan itu
- **Default values:** Pertimbangkan apakah kolom Boolean perlu `DEFAULT` value atau bisa `NULL`

## 5. Kesimpulan

PostgreSQL memiliki implementasi Boolean yang **native dan efisien**:

- ✅ Tipe data sesungguhnya (bukan alias integer)
- ✅ Hanya **1 byte** storage (paling efisien)
- ✅ Menerima banyak format input (true/false, 1/0, yes/no, on/off, t/f)
- ✅ Automatic conversion ke representasi Boolean internal
- ✅ Mendukung three-valued logic (true, false, null)

**Prinsip utama:** Gunakan tipe data yang **paling tepat** merepresentasikan data Anda. Untuk data true/false, gunakan Boolean—bukan integer yang di-abuse. Ini membuat kode lebih jelas, lebih type-safe, dan lebih efisien dalam storage.
