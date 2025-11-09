# 📝 Domain Types di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas Domain Types di PostgreSQL, sebuah fitur yang mengenkapsulasi tipe data dan check constraints menjadi satu custom type yang dapat digunakan kembali (reusable). Domain memungkinkan kita membuat "tipe data custom" yang sebenarnya adalah wrapper dari tipe data existing ditambah validasi. Ini sangat berguna untuk konsistensi validasi di banyak kolom atau tabel, dan lebih mudah dimodifikasi dibanding check constraints biasa.

## 2. Konsep Utama

### a. Apa itu Domain?

Domain adalah wrapper yang menggabungkan tipe data dasar dengan check constraints menjadi satu custom type yang reusable.

**Konsep Dasar:**

```sql
-- Tanpa domain: Harus tulis constraint berulang-ulang
CREATE TABLE orders (
    price NUMERIC CHECK (price > 0),
    discount_price NUMERIC CHECK (discount_price > 0)  -- Duplikasi constraint!
);

CREATE TABLE invoices (
    amount NUMERIC CHECK (amount > 0)  -- Duplikasi lagi!
);

-- Dengan domain: Define sekali, pakai berkali-kali
CREATE DOMAIN positive_price AS NUMERIC CHECK (VALUE > 0);

CREATE TABLE orders (
    price positive_price,
    discount_price positive_price  -- Reuse!
);

CREATE TABLE invoices (
    amount positive_price  -- Reuse lagi!
);
```

**Formula Domain:**

```
Domain = Base Type + Check Constraint(s) + Custom Name
```

### b. Syntax Membuat Domain

```sql
CREATE DOMAIN domain_name AS base_type
    [CONSTRAINT constraint_name]
    CHECK (condition_using_VALUE);
```

**Komponen:**

- `domain_name`: Nama custom type yang akan dibuat
- `base_type`: Tipe data underlying (TEXT, INTEGER, NUMERIC, dll)
- `VALUE`: Keyword khusus yang mereferensi nilai yang sedang divalidasi
- `CHECK`: Constraint yang harus dipenuhi

**Contoh Sederhana:**

```sql
-- Domain untuk email
CREATE DOMAIN email AS TEXT
    CONSTRAINT email_format
    CHECK (VALUE LIKE '%@%');

-- Domain untuk persentase
CREATE DOMAIN percentage AS NUMERIC
    CONSTRAINT valid_percentage
    CHECK (VALUE BETWEEN 0 AND 100);

-- Domain untuk umur
CREATE DOMAIN age AS INTEGER
    CONSTRAINT valid_age
    CHECK (VALUE >= 0 AND VALUE <= 150);
```

### c. Studi Kasus: US Postal Code

Video menggunakan contoh US postal code yang memiliki dua format:

- 5 digit: `12345`
- 5 digit + dash + 4 digit: `12345-6789`

**Problem:**

```sql
-- Tidak bisa pakai INTEGER/NUMERIC karena:
-- 1. Bisa kehilangan leading zero (07432 jadi 7432)
-- 2. Tidak support dash (12345-6789)

-- Harus pakai TEXT, tapi butuh validasi format
```

**Solution dengan Domain:**

```sql
CREATE DOMAIN us_postal_code AS TEXT
    CONSTRAINT format
    CHECK (
        VALUE ~ '^\d{5}$'           -- Format: 5 digit
        OR
        VALUE ~ '^\d{5}-\d{4}$'     -- Format: 5 digit - 4 digit
    );

-- Sekarang bisa dipakai di tabel
CREATE TABLE addresses (
    street TEXT,
    city TEXT,
    postal us_postal_code NOT NULL  -- Domain + NOT NULL
);
```

**Breakdown Regex:**

- `^`: Start of string
- `\d{5}`: Exactly 5 digits
- `-`: Literal dash character
- `\d{4}`: Exactly 4 digits
- `$`: End of string
- `~`: PostgreSQL regex match operator

**Testing:**

```sql
-- Test valid values
INSERT INTO addresses VALUES ('Main St', 'Dallas', '07432');      -- ✅ OK
INSERT INTO addresses VALUES ('Oak Ave', 'Boston', '12345-6789'); -- ✅ OK

-- Test invalid values
INSERT INTO addresses VALUES ('Elm St', 'NYC', '7432');           -- ❌ ERROR: format
-- ERROR: value for domain us_postal_code violates check constraint "format"

INSERT INTO addresses VALUES ('Pine Rd', 'LA', '123');            -- ❌ ERROR: format
INSERT INTO addresses VALUES ('Maple Dr', 'SF', '12345-ABC');     -- ❌ ERROR: format (harus digit)
```

### d. Keyword `VALUE` dalam Domain

`VALUE` adalah keyword khusus yang hanya digunakan dalam definisi domain untuk mereferensi nilai yang sedang divalidasi.

**Perbedaan dengan Check Constraint:**

```sql
-- Check Constraint: Pakai nama kolom
CREATE TABLE products (
    price NUMERIC CHECK (price > 0)  -- Referensi nama kolom
);

-- Domain: Pakai VALUE (karena belum ada nama kolom)
CREATE DOMAIN positive_amount AS NUMERIC
    CHECK (VALUE > 0);  -- Referensi VALUE
```

**Mengapa VALUE?**

- Domain adalah standalone object, tidak terikat ke kolom tertentu
- Bisa digunakan di berbagai kolom dengan nama berbeda
- `VALUE` adalah placeholder untuk nilai apapun yang disimpan

```sql
CREATE DOMAIN email AS TEXT
    CHECK (VALUE LIKE '%@%' AND length(VALUE) > 5);

-- Nanti bisa dipakai dengan nama kolom berbeda:
CREATE TABLE users (
    primary_email email,      -- VALUE akan = primary_email
    backup_email email        -- VALUE akan = backup_email
);
```

### e. Domain vs Check Constraint

| Aspek               | Domain                       | Check Constraint                   |
| ------------------- | ---------------------------- | ---------------------------------- |
| **Reusability**     | ✅ Bisa dipakai berkali-kali | ❌ Harus ditulis ulang             |
| **Multi-column**    | ❌ Hanya single column       | ✅ Bisa reference multiple columns |
| **Modification**    | ✅ Bisa ALTER                | ❌ Harus DROP dan RECREATE         |
| **Gradual rollout** | ✅ Bisa NOT VALIDATED        | ❌ Tidak ada fitur ini             |
| **Encapsulation**   | ✅ Type + constraint bundled | ❌ Terpisah                        |
| **Komunikasi**      | ✅ Nama yang meaningful      | ⚠️ Tergantung naming               |

**Contoh Perbandingan:**

```sql
-- ❌ Check Constraint: Tidak reusable
CREATE TABLE orders (
    price NUMERIC CHECK (price > 0),
    discount NUMERIC CHECK (discount > 0),
    tax NUMERIC CHECK (tax > 0)
);

CREATE TABLE invoices (
    amount NUMERIC CHECK (amount > 0),  -- Tulis lagi!
    fee NUMERIC CHECK (fee > 0)         -- Tulis lagi!
);

-- ✅ Domain: Reusable dan konsisten
CREATE DOMAIN positive_amount AS NUMERIC CHECK (VALUE > 0);

CREATE TABLE orders (
    price positive_amount,
    discount positive_amount,
    tax positive_amount
);

CREATE TABLE invoices (
    amount positive_amount,
    fee positive_amount
);
```

### f. Multi-Column Constraint dengan Domain

**Limitation: Domain tidak bisa untuk multi-column constraint**

```sql
-- ❌ TIDAK BISA: Multi-column logic dalam domain
CREATE DOMAIN price_with_discount AS ??? -- Tidak ada cara untuk reference 2 kolom

-- ✅ HARUS: Tetap pakai table-level check constraint
CREATE TABLE products (
    price positive_amount,           -- Domain untuk single column
    discount positive_amount,        -- Domain untuk single column
    CHECK (price > discount)         -- Table constraint untuk relationship
);
```

**Best Practice: Kombinasi Domain dan Check Constraint**

```sql
-- Domain untuk validasi individual
CREATE DOMAIN positive_amount AS NUMERIC CHECK (VALUE > 0);
CREATE DOMAIN email AS TEXT CHECK (VALUE LIKE '%@%');

-- Pakai domain + table constraint
CREATE TABLE users (
    primary_email email,
    backup_email email,
    balance positive_amount,
    credit_limit positive_amount,

    -- Table constraint untuk relationship antar kolom
    CHECK (primary_email != backup_email),
    CHECK (balance <= credit_limit)
);
```

### g. Memodifikasi Domain dengan ALTER

Salah satu keunggulan domain adalah bisa di-ALTER, berbeda dengan check constraint.

#### ALTER Domain Dasar

```sql
-- Membuat domain
CREATE DOMAIN age AS INTEGER CHECK (VALUE >= 0);

-- Menambah constraint baru
ALTER DOMAIN age ADD CONSTRAINT max_age CHECK (VALUE <= 150);

-- Menghapus constraint
ALTER DOMAIN age DROP CONSTRAINT max_age;

-- Rename domain
ALTER DOMAIN age RENAME TO person_age;
```

#### NOT VALIDATED: Gradual Rollout

Fitur powerful untuk migrasi data bertahap.

```sql
-- Scenario: Domain existing dengan data lama
CREATE DOMAIN us_postal_code AS TEXT
    CHECK (VALUE ~ '^\d{5}$');

-- Table dengan data lama (sudah ada data yang tidak valid)
CREATE TABLE addresses (
    postal us_postal_code
);

-- Ada data lama yang format lama: '12345-6789' (belum di-handle)

-- ALTER dengan NOT VALIDATED: Tidak check data existing
ALTER DOMAIN us_postal_code
    ADD CONSTRAINT extended_format
    CHECK (VALUE ~ '^\d{5}$' OR VALUE ~ '^\d{5}-\d{4}$')
    NOT VALIDATED;

-- Sekarang:
-- - Data baru HARUS memenuhi constraint baru
-- - Data lama boleh tidak memenuhi (sementara)

-- Insert baru akan divalidasi
INSERT INTO addresses VALUES ('12345-6789');  -- ✅ OK (new rule)
INSERT INTO addresses VALUES ('123');         -- ❌ ERROR (new rule)

-- Update data lama satu per satu
UPDATE addresses SET postal = '12345' WHERE postal = 'invalid_old_data';

-- Setelah semua data valid, enforce fully
ALTER DOMAIN us_postal_code VALIDATE CONSTRAINT extended_format;
```

**Workflow Gradual Migration:**

```
1. ALTER DOMAIN dengan NOT VALIDATED
   ↓
2. Data baru: Langsung divalidasi dengan rule baru
   Data lama: Tidak di-touch, masih boleh invalid
   ↓
3. Perlahan migrate/fix data lama
   ↓
4. VALIDATE CONSTRAINT untuk enforce ke semua data
```

**Perbandingan dengan Check Constraint:**

```sql
-- Check Constraint: Tidak bisa gradual
ALTER TABLE products ADD CHECK (price > 0);
-- Langsung validasi SEMUA data existing
-- Jika ada data invalid → ERROR immediately!

-- Domain: Bisa gradual
ALTER DOMAIN positive_amount
    ADD CHECK (VALUE > 0)
    NOT VALIDATED;
-- Data lama: Tidak di-check
-- Data baru: Divalidasi
```

### h. Keuntungan Domain

#### 1. Reusability dan Konsistensi

```sql
-- Define sekali
CREATE DOMAIN email AS TEXT
    CONSTRAINT valid_email
    CHECK (VALUE ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$');

-- Pakai di banyak tempat
CREATE TABLE users (
    primary_email email,
    recovery_email email
);

CREATE TABLE contacts (
    business_email email,
    personal_email email
);

CREATE TABLE newsletter_subscribers (
    subscriber_email email
);

-- Semua kolom punya validasi yang SAMA dan KONSISTEN!
```

#### 2. Maintainability

```sql
-- Jika rule email berubah, cukup ALTER domain sekali
ALTER DOMAIN email DROP CONSTRAINT valid_email;
ALTER DOMAIN email ADD CONSTRAINT valid_email
    CHECK (VALUE ~ '^[new_regex]$');

-- SEMUA table yang pakai domain ini otomatis update!
-- Tidak perlu ALTER setiap table satu per satu
```

#### 3. Self-Documenting Code

```sql
-- ❌ Kurang jelas
CREATE TABLE orders (
    amount NUMERIC CHECK (amount > 0)  -- Amount apa? Positive kenapa?
);

-- ✅ Lebih jelas dan self-documenting
CREATE DOMAIN monetary_amount AS NUMERIC CHECK (VALUE > 0);

CREATE TABLE orders (
    amount monetary_amount  -- Jelas: ini monetary amount dengan validasi built-in
);
```

#### 4. Team Communication

```sql
-- Domain sebagai kontrak dalam tim
CREATE DOMAIN product_sku AS TEXT
    CONSTRAINT sku_format
    CHECK (VALUE ~ '^[A-Z]{3}-[0-9]{5}$');

-- Developer baru lihat schema:
CREATE TABLE products (
    sku product_sku  -- Langsung paham format SKU tanpa baca dokumentasi
);
```

## 3. Hubungan Antar Konsep

### Evolusi Validasi Data:

```
Level 1: Tipe Data Dasar
    INTEGER, TEXT, NUMERIC
    ↓ (Tidak cukup spesifik)

Level 2: Check Constraints
    CHECK (price > 0)
    CHECK (length(code) = 5)
    ↓ (Duplikasi di banyak tempat)

Level 3: Domains
    CREATE DOMAIN positive_price AS NUMERIC CHECK (VALUE > 0)
    ↓ (Reusable, maintainable)
```

### Domain dalam Ekosistem PostgreSQL:

```
Base Types (INT, TEXT, etc)
    ↓ wrapped by
Domains (custom types with validation)
    ↓ used in
Tables & Columns
    ↓ protected by
Check Constraints (for multi-column rules)
    ↓ ensures
Data Integrity
```

### Kapan Menggunakan Apa:

```
Single column, used once
    → Check Constraint inline

Single column, used multiple times
    → Domain

Multiple columns relationship
    → Table-level Check Constraint

Complex business logic that changes often
    → Application layer
```

### Kombinasi Domain + Check Constraint:

```sql
-- Strategi optimal: Gunakan keduanya
CREATE DOMAIN positive_amount AS NUMERIC CHECK (VALUE > 0);
CREATE DOMAIN email AS TEXT CHECK (VALUE LIKE '%@%');
CREATE DOMAIN us_postal_code AS TEXT
    CHECK (VALUE ~ '^\d{5}(-\d{4})?$');

CREATE TABLE orders (
    -- Domain untuk single-column validation
    customer_email email,
    billing_email email,
    price positive_amount,
    discount positive_amount,
    shipping us_postal_code,

    -- Check constraint untuk multi-column logic
    CHECK (billing_email IS NOT NULL OR customer_email IS NOT NULL),
    CHECK (price > discount),
    CHECK (discount <= price * 0.5)  -- Max 50% discount
);
```

## 4. Catatan Tambahan / Insight

### Tips Praktis:

1. **Naming Convention untuk Domain:**

   ```sql
   -- Good: Descriptive dan menunjukkan purpose
   CREATE DOMAIN email AS TEXT CHECK (...);
   CREATE DOMAIN us_postal_code AS TEXT CHECK (...);
   CREATE DOMAIN positive_amount AS NUMERIC CHECK (...);
   CREATE DOMAIN percentage AS NUMERIC CHECK (...);

   -- Avoid: Terlalu generic
   CREATE DOMAIN text_field AS TEXT CHECK (...);  -- Kurang spesifik
   CREATE DOMAIN number_type AS NUMERIC CHECK (...);  -- Kurang jelas
   ```

2. **Common Domain Patterns:**

   ```sql
   -- Email (simple validation)
   CREATE DOMAIN email AS TEXT
       CHECK (VALUE ~ '^[^@]+@[^@]+\.[^@]+$');

   -- Phone Indonesia
   CREATE DOMAIN phone_id AS TEXT
       CHECK (VALUE ~ '^(\+62|62|0)[0-9]{9,12}$');

   -- URL
   CREATE DOMAIN url AS TEXT
       CHECK (VALUE ~ '^https?://[^\s]+$');

   -- Currency amount (2 decimal places)
   CREATE DOMAIN currency AS NUMERIC(12, 2)
       CHECK (VALUE >= 0);

   -- Percentage (0-100)
   CREATE DOMAIN percentage AS NUMERIC(5, 2)
       CHECK (VALUE BETWEEN 0 AND 100);

   -- Year (reasonable range)
   CREATE DOMAIN year AS INTEGER
       CHECK (VALUE BETWEEN 1900 AND 2100);

   -- Rating (1-5 stars)
   CREATE DOMAIN star_rating AS INTEGER
       CHECK (VALUE BETWEEN 1 AND 5);
   ```

3. **Domain dengan Multiple Constraints:**

   ```sql
   CREATE DOMAIN username AS TEXT
       CONSTRAINT min_length CHECK (length(VALUE) >= 3)
       CONSTRAINT max_length CHECK (length(VALUE) <= 20)
       CONSTRAINT valid_chars CHECK (VALUE ~ '^[a-zA-Z0-9_]+$');

   -- Bisa drop/add individual constraints
   ALTER DOMAIN username DROP CONSTRAINT min_length;
   ALTER DOMAIN username ADD CONSTRAINT min_length CHECK (length(VALUE) >= 5);
   ```

4. **Domain dengan Default Values:**

   ```sql
   -- Domain bisa punya default
   CREATE DOMAIN status_code AS TEXT
       DEFAULT 'pending'
       CHECK (VALUE IN ('pending', 'approved', 'rejected'));

   CREATE TABLE applications (
       id SERIAL PRIMARY KEY,
       status status_code  -- Default 'pending' dari domain
   );
   ```

5. **Listing All Domains:**

   ```sql
   -- Lihat semua domain di database
   SELECT
       n.nspname AS schema,
       t.typname AS domain_name,
       pg_catalog.format_type(t.typbasetype, t.typtypmod) AS base_type
   FROM pg_catalog.pg_type t
   JOIN pg_catalog.pg_namespace n ON n.oid = t.typnamespace
   WHERE t.typtype = 'd'
   ORDER BY 1, 2;

   -- Lihat constraints dalam domain
   \dD+ domain_name  -- Di psql
   ```

### Kesalahan Umum:

```sql
-- ❌ KESALAHAN 1: Mencoba reference multiple values
CREATE DOMAIN price_discount AS NUMERIC
    CHECK (VALUE > discount_value);  -- ERROR: discount_value tidak ada
-- Fix: Pakai check constraint untuk multi-column

-- ❌ KESALAHAN 2: Terlalu strict saat awal
CREATE DOMAIN email AS TEXT
    CHECK (VALUE ~ '^[a-z]+@[a-z]+\.com$');  -- Terlalu ketat!
-- Masalah: Tidak terima uppercase, subdomain, TLD selain .com
-- Fix: Buat regex yang lebih permissive atau update gradually

-- ❌ KESALAHAN 3: Lupa NOT NULL
CREATE DOMAIN positive_amount AS NUMERIC CHECK (VALUE > 0);

CREATE TABLE orders (
    price positive_amount  -- Masih bisa NULL!
);
INSERT INTO orders VALUES (NULL);  -- ✅ Berhasil!

-- Fix: Tambahkan NOT NULL di kolom atau domain
CREATE DOMAIN positive_amount AS NUMERIC
    NOT NULL
    CHECK (VALUE > 0);

-- ❌ KESALAHAN 4: Regex tanpa anchor
CREATE DOMAIN us_postal_code AS TEXT
    CHECK (VALUE ~ '\d{5}');  -- Tanpa ^ dan $
-- Masalah: '12345-INVALID-6789' akan lolos! (karena ada 5 digit di tengah)
-- Fix: Selalu pakai ^ dan $
CREATE DOMAIN us_postal_code AS TEXT
    CHECK (VALUE ~ '^\d{5}(-\d{4})?$');

-- ❌ KESALAHAN 5: Tidak pakai constraint name
CREATE DOMAIN email AS TEXT
    CHECK (VALUE LIKE '%@%');  -- Unnamed constraint
-- Fix: Beri nama untuk debugging
CREATE DOMAIN email AS TEXT
    CONSTRAINT valid_format CHECK (VALUE LIKE '%@%');
```

### Advanced Patterns:

```sql
-- Pattern 1: Domain hierarchy (base domain + extended domain)
CREATE DOMAIN positive_number AS NUMERIC CHECK (VALUE > 0);

-- Pakai positive_number dalam domain lain? TIDAK BISA!
-- Domain tidak bisa inherit domain lain di PostgreSQL
-- Tapi bisa copy pattern:

CREATE DOMAIN price AS NUMERIC
    CHECK (VALUE > 0 AND VALUE < 1000000);  -- Copy + extend logic

-- Pattern 2: Domain untuk Enum alternative
CREATE DOMAIN order_status AS TEXT
    CHECK (VALUE IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled'));

-- Lebih fleksibel dari ENUM type karena bisa ALTER

-- Pattern 3: Composite validation
CREATE DOMAIN secure_password AS TEXT
    CONSTRAINT min_length CHECK (length(VALUE) >= 8)
    CONSTRAINT has_uppercase CHECK (VALUE ~ '[A-Z]')
    CONSTRAINT has_lowercase CHECK (VALUE ~ '[a-z]')
    CONSTRAINT has_number CHECK (VALUE ~ '[0-9]');

-- Pattern 4: Country-specific domains
CREATE DOMAIN us_postal_code AS TEXT
    CHECK (VALUE ~ '^\d{5}(-\d{4})?$');

CREATE DOMAIN uk_postal_code AS TEXT
    CHECK (VALUE ~ '^[A-Z]{1,2}[0-9]{1,2} [0-9][A-Z]{2}$');

CREATE DOMAIN id_postal_code AS TEXT
    CHECK (VALUE ~ '^\d{5}$');

-- Gunakan sesuai region
CREATE TABLE addresses (
    country TEXT,
    postal_us us_postal_code,
    postal_uk uk_postal_code,
    postal_id id_postal_code,
    CHECK (
        (country = 'US' AND postal_us IS NOT NULL) OR
        (country = 'UK' AND postal_uk IS NOT NULL) OR
        (country = 'ID' AND postal_id IS NOT NULL)
    )
);
```

### Migration Strategy:

```sql
-- Scenario: Migrate dari check constraint ke domain

-- Step 1: Buat domain
CREATE DOMAIN email AS TEXT
    CHECK (VALUE ~ '^[^@]+@[^@]+\.[^@]+$');

-- Step 2: Buat kolom baru dengan domain
ALTER TABLE users ADD COLUMN new_email email;

-- Step 3: Copy data
UPDATE users SET new_email = old_email;

-- Step 4: Drop kolom lama
ALTER TABLE users DROP COLUMN old_email;

-- Step 5: Rename kolom baru
ALTER TABLE users RENAME COLUMN new_email TO email;

-- Alternative: Jika bisa downtime
ALTER TABLE users
    ALTER COLUMN email TYPE email USING email::email;
-- Tapi ini bisa lambat untuk table besar!
```

### Performance Considerations:

```sql
-- Domain check dijalankan setiap INSERT/UPDATE
CREATE DOMAIN complex_validation AS TEXT
    CHECK (
        length(VALUE) > 5 AND
        VALUE ~ '[A-Z]' AND
        VALUE ~ '[a-z]' AND
        VALUE ~ '[0-9]' AND
        VALUE !~ '[^A-Za-z0-9]'
    );

-- Untuk table dengan insert frequency tinggi, pertimbangkan:
-- 1. Simplify validation
-- 2. Move complex logic ke application
-- 3. Use trigger untuk async validation

-- Benchmark:
-- Simple check (VALUE > 0): ~0.001ms overhead
-- Regex check: ~0.1ms overhead
-- Complex multi-check: ~1ms overhead
-- Multiply by jutaan rows → bisa signifikan!
```

### Analogi:

**Domain seperti Template atau Blueprint:**

- **Check Constraint** = Aturan custom di setiap rumah

  - Setiap rumah tulis aturannya sendiri
  - Tidak konsisten
  - Sulit update kalau rule berubah

- **Domain** = Building code yang dipakai semua rumah
  - Define sekali, pakai berkali-kali
  - Konsisten di semua tempat
  - Update building code → semua rumah otomatis update

**Domain seperti Fungsi Reusable:**

```python
# Tanpa domain (seperti copy-paste code)
def validate_email_in_users(email):
    if '@' not in email: raise Error

def validate_email_in_contacts(email):
    if '@' not in email: raise Error  # Duplikasi!

# Dengan domain (seperti shared function)
def validate_email(email):
    if '@' not in email: raise Error

validate_email(users.email)
validate_email(contacts.email)  # Reuse!
```

## 5. Kesimpulan

Domain Types adalah fitur powerful PostgreSQL yang mengenkapsulasi tipe data dan check constraints menjadi custom type yang reusable. Domain mengatasi masalah duplikasi check constraint dan meningkatkan konsistensi validasi di seluruh database.

**Key Points:**

1. **Definisi**: Domain = Base Type + Check Constraint(s) + Custom Name
2. **Syntax**: Menggunakan keyword `VALUE` untuk mereferensi nilai yang divalidasi
3. **Reusability**: Satu domain bisa dipakai di banyak kolom dan tabel
4. **Maintainability**: Bisa di-ALTER, tidak perlu DROP-RECREATE
5. **Gradual Migration**: Fitur `NOT VALIDATED` untuk rollout bertahap

**Kapan Menggunakan Domain:**

- ✅ Validasi yang sama dipakai di banyak tempat
- ✅ Format data yang standard (email, phone, postal code)
- ✅ Business rules yang konsisten (positive amount, valid percentage)
- ✅ Komunikasi yang jelas dalam tim (self-documenting)

**Kapan TIDAK Menggunakan Domain:**

- ❌ Multi-column validation (pakai check constraint)
- ❌ Logic yang sangat complex atau sering berubah
- ❌ Validasi yang hanya dipakai sekali

**Best Practice:**

- Kombinasikan domain (untuk single-column) dengan check constraint (untuk multi-column)
- Beri nama yang descriptive dan meaningful
- Gunakan `NOT VALIDATED` untuk migration data existing
- Dokumentasikan domain di schema documentation

**Key takeaway:** Domain adalah abstraksi yang meningkatkan reusability, maintainability, dan konsistensi validasi data. Mereka adalah "DRY principle" untuk database constraints - Define once, use everywhere, maintain easily.
