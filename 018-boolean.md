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

## 4. Kesimpulan

PostgreSQL memiliki implementasi Boolean yang **native dan efisien**:

- ✅ Tipe data sesungguhnya (bukan alias integer)
- ✅ Hanya **1 byte** storage (paling efisien)
- ✅ Menerima banyak format input (true/false, 1/0, yes/no, on/off, t/f)
- ✅ Automatic conversion ke representasi Boolean internal
- ✅ Mendukung three-valued logic (true, false, null)

**Prinsip utama:** Gunakan tipe data yang **paling tepat** merepresentasikan data Anda. Untuk data true/false, gunakan Boolean—bukan integer yang di-abuse. Ini membuat kode lebih jelas, lebih type-safe, dan lebih efisien dalam storage.
