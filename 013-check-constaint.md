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
    sku TEXT CHECK (length(sku) = 8),             -- SKU harus 8 karakter
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

## 4. Kesimpulan

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
- Kombinasikan dengan NOT NULL jika tidak mau terima NULL (check constraint masih bisa menerima NULL)
- Pertimbangkan performa untuk checks yang complex

**Philosophy:**
Ada perdebatan tentang seberapa banyak logic yang masuk database. Untuk **data integrity** (seperti "harga harus positif", "email harus valid", "panjang harus tepat 5"), check constraints adalah tempat yang tepat. Untuk **business logic** yang complex dan sering berubah, pertimbangkan aplikasi layer.

**Key takeaway:** Check constraints mengisi gap antara tipe data dasar dan kebutuhan validasi bisnis. Mereka adalah layer pertahanan pertama untuk data quality, terutama ketika multiple aplikasi atau tools mengakses database yang sama.
