# 📝 Check Constraints di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang check constraints di PostgreSQL, sebuah mekanisme untuk menerapkan aturan validasi data langsung di level database. Check constraints memungkinkan kita untuk membatasi nilai yang bisa disimpan dalam kolom, seperti memastikan harga selalu positif atau panjang string harus tepat 5 karakter. Video juga membahas perbedaan antara column constraint dan table constraint, serta perdebatan tentang seberapa banyak business logic yang sebaiknya ada di database.

## 2. Konsep Utama

### a. Masalah yang Diselesaikan oleh Check Constraints

Check constraints mengatasi keterbatasan tipe data bawaan PostgreSQL:

**Problem 1: Integer Tidak Punya Unsigned**

```sql
-- Di database lain (MySQL, dll):
CREATE TABLE products (
    quantity UNSIGNED INTEGER  -- Range: 0 sampai 4.2 miliar
);

-- Di PostgreSQL, INTEGER selalu signed:
CREATE TABLE products (
    quantity INTEGER  -- Range: -2.1 miliar sampai +2.1 miliar
);
-- Kita "kehilangan" setengah range untuk nilai negatif yang tidak terpakai!
```

**Problem 2: CHAR(n) Tidak Enforce Panjang yang Tepat**

```sql
-- CHAR(5) tidak memaksa input harus 5 karakter
CREATE TABLE codes (
    abbreviation CHAR(5)
);

INSERT INTO codes VALUES ('A');  -- Berhasil! (jadi 'A    ')
-- Tapi kita mau HARUS 5 karakter, bukan padding!
```

**Solusi: Check Constraints**

```sql
-- Memaksa nilai >= 0
CREATE TABLE products (
    quantity INTEGER CHECK (quantity >= 0)
);

-- Memaksa panjang string tepat 5 karakter
CREATE TABLE codes (
    abbreviation TEXT CHECK (length(abbreviation) = 5)
);
```

### b. Syntax Dasar Check Constraint

#### 1. Column Constraint (Inline)

Check constraint yang langsung dideklarasikan setelah tipe data kolom.

```sql
-- Syntax paling sederhana
CREATE TABLE example (
    price NUMERIC CHECK (price > 0)
);

-- Test: Insert valid data
INSERT INTO example VALUES (10);  -- ✅ Berhasil

-- Test: Insert invalid data
INSERT INTO example VALUES (-1);   -- ❌ ERROR!
-- ERROR: new row violates check constraint "example_price_check"
```

#### 2. Named Constraint (dengan nama custom)

Memberikan nama yang descriptive untuk memudahkan debugging.

```sql
CREATE TABLE example (
    price NUMERIC CONSTRAINT price_must_be_positive CHECK (price > 0)
);

-- Test invalid data
INSERT INTO example VALUES (-1);
-- ERROR: new row violates check constraint "price_must_be_positive"
-- Nama constraint lebih jelas dan membantu debugging!
```

**Keuntungan memberi nama:**

- Error message lebih jelas
- Memudahkan komunikasi dalam tim
- Lebih informatif di log file
- Memudahkan maintenance

```sql
-- Contoh lengkap dengan multiple constraints
CREATE TABLE example (
    price NUMERIC CONSTRAINT price_must_be_positive CHECK (price > 0),
    abbreviation TEXT CONSTRAINT abbr_length_five CHECK (length(abbreviation) = 5)
);

-- Test
INSERT INTO example VALUES (10, 'ABCDE');  -- ✅ OK
INSERT INTO example VALUES (10, 'ABC');    -- ❌ ERROR: abbr_length_five
INSERT INTO example VALUES (-5, 'ABCDE');  -- ❌ ERROR: price_must_be_positive
```

### c. Column Constraint vs Table Constraint

#### Column Constraint

Dideklarasikan langsung setelah definisi kolom, hanya bisa mereferensi satu kolom.

```sql
CREATE TABLE example (
    price NUMERIC CHECK (price > 0),              -- Column constraint
    abbreviation TEXT CHECK (length(abbreviation) = 5)  -- Column constraint
);
```

**Karakteristik:**

- ✅ Ringkas untuk constraint sederhana
- ✅ Mudah dibaca untuk single-column rules
- ❌ Hanya bisa referensi kolom itu sendiri

#### Table Constraint

Dideklarasikan terpisah setelah semua kolom, bisa mereferensi multiple kolom.

```sql
CREATE TABLE example (
    price NUMERIC,
    abbreviation TEXT,
    -- Table constraints di sini
    CHECK (price > 0),
    CHECK (length(abbreviation) = 5)
);
```

**Kapan HARUS menggunakan Table Constraint:**
Ketika constraint melibatkan lebih dari satu kolom.

```sql
CREATE TABLE example (
    price NUMERIC,
    discount_price NUMERIC,
    abbreviation TEXT,

    -- Column constraints (opsional di sini atau inline)
    CHECK (price > 0),
    CHECK (discount_price > 0),
    CHECK (length(abbreviation) = 5),

    -- Table constraint (HARUS di sini karena melibatkan 2 kolom)
    CHECK (price > discount_price)
);
```

**Best Practice:**

```sql
-- ❌ BURUK: Multi-column constraint di column level
CREATE TABLE products (
    price NUMERIC CHECK (price > discount_price),  -- BAD! Belum ada discount_price
    discount_price NUMERIC
);

-- ✅ BAIK: Multi-column constraint di table level
CREATE TABLE products (
    price NUMERIC,
    discount_price NUMERIC,
    CHECK (price > discount_price)
);
```

### d. Contoh Lengkap dengan Multiple Constraints

```sql
CREATE TABLE products (
    price NUMERIC CONSTRAINT price_positive CHECK (price > 0),
    discount_price NUMERIC CONSTRAINT discount_positive CHECK (discount_price > 0),
    abbreviation TEXT,

    -- Multi-column constraint
    CONSTRAINT price_greater_than_discount CHECK (price > discount_price),
    CHECK (length(abbreviation) = 5)
);

-- Test case 1: Semua valid
INSERT INTO products VALUES (10, 8, 'ABCDE');  -- ✅ OK

-- Test case 2: discount_price negatif
INSERT INTO products VALUES (10, -8, 'ABCDE'); -- ❌ ERROR: discount_positive

-- Test case 3: price negatif
INSERT INTO products VALUES (-10, 8, 'ABCDE'); -- ❌ ERROR: price_positive

-- Test case 4: price < discount_price
INSERT INTO products VALUES (8, 10, 'ABCDE');  -- ❌ ERROR: price_greater_than_discount

-- Test case 5: abbreviation tidak 5 karakter
INSERT INTO products VALUES (10, 8, 'ABC');    -- ❌ ERROR: length check
```

### e. Batasan Check Constraints

#### 1. Tidak Bisa Referensi Tabel Lain

```sql
-- ❌ TIDAK BISA: Referensi tabel lain
CREATE TABLE orders (
    product_id INTEGER,
    quantity INTEGER,
    CHECK (quantity <= (SELECT stock FROM products WHERE id = product_id))
);
-- ERROR: cannot use subquery in check constraint

-- ✅ ALTERNATIF: Gunakan trigger atau foreign key
```

#### 2. Tidak Bisa Referensi Row Lain dalam Tabel yang Sama

```sql
-- ❌ TIDAK BISA: Membandingkan dengan row lain
CREATE TABLE inventory (
    item_id INTEGER,
    quantity INTEGER,
    CHECK (quantity < (SELECT MAX(quantity) FROM inventory))
);
-- ERROR: cannot use subquery in check constraint

-- Check constraint hanya bisa akses nilai dalam row yang sedang di-insert/update
```

#### 3. Hanya Menerima Row Being Inserted/Updated

```sql
-- ✅ BISA: Akses nilai dalam row yang sama
CREATE TABLE orders (
    price NUMERIC,
    tax NUMERIC,
    total NUMERIC,
    CHECK (total = price + tax)  -- OK, semua dalam row yang sama
);
```

### f. Memodifikasi Check Constraints

#### Tidak Bisa ALTER, Harus DROP dan RECREATE

```sql
-- ❌ TIDAK BISA: Alter constraint logic
ALTER TABLE products ALTER CONSTRAINT price_positive CHECK (price > 10);
-- ERROR: syntax error

-- ✅ HARUS: Drop dan recreate
-- Cara 1: Dua statement terpisah (ada window tanpa constraint!)
ALTER TABLE products DROP CONSTRAINT price_positive;
ALTER TABLE products ADD CONSTRAINT price_positive CHECK (price > 10);

-- Cara 2: Single statement (lebih aman!)
ALTER TABLE products
    DROP CONSTRAINT price_positive,
    ADD CONSTRAINT price_positive CHECK (price > 10);
```

**Kenapa single statement lebih baik:**

```sql
-- Scenario dengan dua statement:
ALTER TABLE products DROP CONSTRAINT price_positive;
-- ⚠️ Window waktu tanpa constraint!
-- User bisa insert data invalid di sini!
ALTER TABLE products ADD CONSTRAINT price_positive CHECK (price > 10);

-- Single statement: Atomic operation
ALTER TABLE products
    DROP CONSTRAINT price_positive,
    ADD CONSTRAINT price_positive CHECK (price > 10);
-- ✅ Tidak ada window tanpa constraint
```

### g. Business Logic vs Data Integrity

Video memberikan perspektif tentang perdebatan klasik: Seberapa banyak logic yang sebaiknya ada di database?

**Data Integrity (✅ Cocok untuk Database):**

```sql
-- Constraint yang melindungi konsistensi data
CREATE TABLE products (
    price NUMERIC CHECK (price > 0),              -- Harga harus positif
    stock INTEGER CHECK (stock >= 0),             -- Stok tidak boleh negatif
    sku TEXT CHECK (length(sku) = 8),            -- SKU harus 8 karakter
    discount_price NUMERIC,
    CHECK (price >= discount_price)               -- Harga > harga diskon
);
```

**Business Logic (⚠️ Pertimbangkan Aplikasi Layer):**

```sql
-- Complex calculations atau rules yang sering berubah
CREATE TABLE orders (
    subtotal NUMERIC,
    discount NUMERIC,
    -- ⚠️ Terlalu complex untuk constraint?
    CHECK (
        CASE
            WHEN subtotal > 1000 THEN discount <= subtotal * 0.2
            WHEN subtotal > 500 THEN discount <= subtotal * 0.1
            ELSE discount <= subtotal * 0.05
        END
    )
);
-- Mungkin lebih baik di aplikasi karena rule bisa sering berubah
```

**Rekomendasi:**

| Kriteria                              | Database (CHECK) | Application |
| ------------------------------------- | ---------------- | ----------- |
| Data yang konsisten secara struktural | ✅               |             |
| Rule yang jarang berubah              | ✅               |             |
| Multiple aplikasi akses DB            | ✅               |             |
| Complex business calculations         |                  | ✅          |
| Rule yang sering berubah              |                  | ✅          |
| Membutuhkan external data             |                  | ✅          |

**Contoh yang JELAS Database:**

```sql
-- Ini bukan "business logic", ini data integrity
CREATE TABLE users (
    age INTEGER CHECK (age >= 0 AND age <= 150),
    email TEXT CHECK (email LIKE '%@%')
);
```

**Contoh yang MUNGKIN Aplikasi:**

```sql
-- Business rule yang bisa berubah
CREATE TABLE memberships (
    tier TEXT CHECK (tier IN ('bronze', 'silver', 'gold', 'platinum')),
    -- Apa tier-nya berubah? Butuh ALTER constraint
    discount_percent NUMERIC CHECK (
        (tier = 'platinum' AND discount_percent <= 20) OR
        (tier = 'gold' AND discount_percent <= 15) OR
        (tier = 'silver' AND discount_percent <= 10) OR
        (tier = 'bronze' AND discount_percent <= 5)
    )
    -- Rule ini bisa sering berubah, mungkin lebih baik di aplikasi
);
```

## 3. Hubungan Antar Konsep

### Flow Pemilihan Constraint Type:

```
Apakah constraint melibatkan multiple kolom?
├─ Ya → HARUS Table Constraint
│   └─ Contoh: CHECK (price > discount_price)
└─ Tidak → Boleh Column atau Table Constraint (preferensi style)
    ├─ Column Constraint (inline, ringkas)
    │   └─ Contoh: price NUMERIC CHECK (price > 0)
    └─ Table Constraint (terpisah, konsisten dengan multi-column)
        └─ Contoh: CHECK (price > 0)
```

### Evolusi dari Video Sebelumnya:

```
Video: Character Types
    Problem: CHAR(5) tidak enforce panjang tepat 5
    ↓
Video: Check Constraints
    Solution: TEXT + CHECK (length(col) = 5)

Video: Numeric Types
    Problem: Tidak ada UNSIGNED INTEGER
    ↓
Video: Check Constraints
    Solution: INTEGER + CHECK (col >= 0)
```

### Hirarki Data Protection:

```
Level 1: Tipe Data
    ├─ INTEGER (bukan TEXT)
    └─ NUMERIC (bukan VARCHAR)

Level 2: Check Constraints
    ├─ CHECK (price > 0)
    └─ CHECK (length(sku) = 8)

Level 3: Foreign Keys (belum dibahas)
    └─ REFERENCES other_table(id)

Level 4: Triggers (complex logic)
    └─ BEFORE INSERT/UPDATE
```

## 4. Catatan Tambahan / Insight

### Tips Praktis:

1. **Naming Convention untuk Constraints:**

   ```sql
   -- Pattern: {table}_{column}_{rule}
   CREATE TABLE products (
       price NUMERIC CONSTRAINT products_price_positive CHECK (price > 0),
       stock INTEGER CONSTRAINT products_stock_nonnegative CHECK (stock >= 0)
   );

   -- Atau lebih descriptive:
   CREATE TABLE products (
       price NUMERIC CONSTRAINT price_must_be_positive CHECK (price > 0),
       discount NUMERIC CONSTRAINT discount_cannot_exceed_price
           CHECK (discount <= price)
   );
   ```

2. **Common Check Patterns:**

   ```sql
   -- Email validation (sederhana)
   email TEXT CHECK (email LIKE '%@%.%')

   -- Phone number (Indonesia)
   phone TEXT CHECK (phone ~ '^[0-9]{10,13}$')

   -- Percentage (0-100)
   discount_pct NUMERIC CHECK (discount_pct BETWEEN 0 AND 100)

   -- Enum alternative
   status TEXT CHECK (status IN ('pending', 'approved', 'rejected'))

   -- Date range
   start_date DATE,
   end_date DATE,
   CHECK (end_date > start_date)

   -- URL validation (sederhana)
   website TEXT CHECK (website LIKE 'http%://%')
   ```

3. **Menangani NULL:**

   ```sql
   -- Check constraint tidak mencegah NULL!
   CREATE TABLE products (
       price NUMERIC CHECK (price > 0)
   );

   INSERT INTO products VALUES (NULL);  -- ✅ BERHASIL!
   -- Check constraint hanya evaluasi ketika nilai NOT NULL

   -- Jika mau prevent NULL:
   CREATE TABLE products (
       price NUMERIC NOT NULL CHECK (price > 0)
   );

   INSERT INTO products VALUES (NULL);  -- ❌ ERROR: NOT NULL violation
   ```

4. **Performance Considerations:**

   ```sql
   -- Simple checks: Sangat cepat
   CHECK (price > 0)  -- ✅ Instant

   -- Complex checks: Bisa lambat
   CHECK (
       price > 0 AND
       discount_price > 0 AND
       price > discount_price AND
       length(description) > 10 AND
       upper(status) IN ('ACTIVE', 'INACTIVE')
   )  -- ⚠️ Dievaluasi setiap INSERT/UPDATE

   -- Function calls: Hati-hati!
   CHECK (expensive_function(column) = true)  -- ⚠️ Bisa sangat lambat
   ```

5. **Debugging Check Constraint Failures:**

   ```sql
   -- Strategi 1: Named constraints
   CONSTRAINT readable_name CHECK (complex_condition)

   -- Strategi 2: Test each condition
   INSERT INTO products VALUES (...);  -- Fails

   -- Test manually:
   SELECT
       price > 0 AS price_ok,
       discount_price > 0 AS discount_ok,
       price > discount_price AS price_vs_discount_ok,
       length(sku) = 8 AS sku_length_ok
   FROM (VALUES (your_test_data)) AS t(...);
   ```

### Kesalahan Umum:

```sql
-- ❌ KESALAHAN 1: Lupa bahwa NULL lolos check
CREATE TABLE products (
    price NUMERIC CHECK (price > 0)
);
INSERT INTO products VALUES (NULL);  -- Berhasil!
-- Fix: Tambahkan NOT NULL

-- ❌ KESALAHAN 2: Check yang terlalu ketat
CREATE TABLE users (
    email TEXT CHECK (email ~ '^[a-z0-9]+@[a-z]+\.[a-z]{2,3}$')
);
-- Masalah: Tidak terima email valid seperti:
-- - john.doe@example.com (ada dot)
-- - admin@sub.example.com (ada subdomain)
-- - user@example.tech (TLD 4 huruf)

-- ❌ KESALAHAN 3: Multi-column check di column level
CREATE TABLE orders (
    price NUMERIC CHECK (price > discount),  -- BAD!
    discount NUMERIC
);
-- Fix: Pindah ke table constraint

-- ❌ KESALAHAN 4: Constraint yang tidak bisa diubah dengan mudah
CREATE TABLE products (
    status TEXT CHECK (status IN ('draft', 'published'))
);
-- Nanti mau tambah status 'archived'? Harus ALTER constraint!
-- Pertimbangkan: Enum type atau reference table

-- ❌ KESALAHAN 5: Complex business logic di constraint
CREATE TABLE orders (
    total NUMERIC,
    customer_tier TEXT,
    CHECK (
        (customer_tier = 'vip' AND total >= 100) OR
        (customer_tier = 'regular' AND total >= 500)
    )
);
-- Ini business logic yang bisa berubah, mungkin lebih baik di aplikasi
```

### Analogi:

**Check Constraint seperti Security Guard:**

- **Tipe Data** = Pintu ukuran standar

  - "Hanya angka boleh masuk"
  - "Hanya teks boleh masuk"

- **Check Constraint** = Security guard dengan checklist

  - "Angka boleh masuk, tapi HARUS positif"
  - "Teks boleh masuk, tapi HARUS 5 karakter"
  - "Harga boleh masuk, tapi HARUS lebih besar dari diskon"

- **NULL** = VIP pass
  - NULL selalu lolos check constraint (kecuali ada NOT NULL)

### Pattern: Migration dari Legacy System

```sql
-- Legacy table: Menggunakan CHAR dan tidak ada validasi
CREATE TABLE legacy_products (
    sku CHAR(8),
    price NUMERIC,
    status CHAR(1)  -- 'A' = active, 'I' = inactive
);

-- Modern approach: TEXT dengan proper constraints
CREATE TABLE modern_products (
    sku TEXT NOT NULL CHECK (length(sku) = 8 AND sku ~ '^[A-Z0-9]+$'),
    price NUMERIC NOT NULL CHECK (price > 0),
    status TEXT NOT NULL CHECK (status IN ('active', 'inactive')),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    -- Table-level constraints
    CONSTRAINT sku_unique UNIQUE (sku)
);

-- Keuntungan modern approach:
-- 1. Tidak ada padding
-- 2. Validasi yang jelas
-- 3. Named constraints untuk debugging
-- 4. Extensible (mudah tambah status baru)
```

### When to Use vs When to Avoid:

**✅ GUNAKAN Check Constraints untuk:**

```sql
-- Data structural integrity
CHECK (age >= 0)
CHECK (quantity >= 0)
CHECK (percentage BETWEEN 0 AND 100)

-- Format validation
CHECK (email LIKE '%@%')
CHECK (length(code) = 6)

-- Logical relationships dalam row yang sama
CHECK (start_date < end_date)
CHECK (price >= discount_price)

-- Enum alternatives (untuk set kecil yang stabil)
CHECK (status IN ('draft', 'published', 'archived'))
```

**⚠️ PERTIMBANGKAN Aplikasi untuk:**

```sql
-- Business rules yang sering berubah
-- Complex calculations
-- Rules yang butuh external data
-- Conditional logic yang complex
-- Rules yang berbeda per customer/region
```

## 5. Kesimpulan

Check constraints adalah tool yang powerful untuk menerapkan data integrity di level database. Mereka memungkinkan kita membuat aturan validasi yang lebih spesifik dari yang bisa dilakukan hanya dengan tipe data saja.

**Key Points:**

1. **Syntax**: Bisa sebagai column constraint (inline) atau table constraint (terpisah)
2. **Naming**: Selalu beri nama untuk memudahkan debugging
3. **Multi-column rules**: HARUS menggunakan table constraint
4. **Limitations**: Tidak bisa referensi tabel lain atau row lain
5. **Modification**: Harus DROP dan RECREATE, preferably dalam single statement

**Best Practices:**

- Gunakan check constraints untuk **data integrity** (bukan complex business logic)
- Beri nama yang descriptive untuk semua constraints
- Pindahkan multi-column constraints ke table level
- Kombinasikan dengan NOT NULL jika tidak mau terima NULL
- Pertimbangkan performa untuk checks yang complex

**Philosophy:**
Ada perdebatan tentang seberapa banyak logic yang masuk database. Untuk **data integrity** (seperti "harga harus positif", "email harus valid", "panjang harus tepat 5"), check constraints adalah tempat yang tepat. Untuk **business logic** yang complex dan sering berubah, pertimbangkan aplikasi layer.

**Key takeaway:** Check constraints mengisi gap antara tipe data dasar dan kebutuhan validasi bisnis. Mereka adalah layer pertahanan pertama untuk data quality, terutama ketika multiple aplikasi atau tools mengakses database yang sama.
