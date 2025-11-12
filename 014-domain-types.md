# 📝 Domain Types di PostgreSQL (VERSI TERKOREKSI)

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

**⚠️ KOREKSI: Domain CAN be based on another domain!**

```sql
-- Base domain
CREATE DOMAIN positive_number AS NUMERIC CHECK (VALUE > 0);

-- Domain based on another domain (derived domain)
CREATE DOMAIN price AS positive_number CHECK (VALUE < 1000000);

-- This is VALID in PostgreSQL!
```

### b. Syntax Membuat Domain

```sql
CREATE DOMAIN domain_name AS base_type
    [CONSTRAINT constraint_name]
    CHECK (condition_using_VALUE);
```

**Komponen:**

- `domain_name`: Nama custom type yang akan dibuat
- `base_type`: Tipe data underlying (TEXT, INTEGER, NUMERIC, dll) - **BISA juga domain lain!**
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
    postal us_postal_code NOT NULL  -- Domain + NOT NULL di KOLOM (best practice!)
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

| Aspek                  | Domain                       | Check Constraint                   |
| ---------------------- | ---------------------------- | ---------------------------------- |
| **Reusability**        | ✅ Bisa dipakai berkali-kali | ❌ Harus ditulis ulang             |
| **Multi-column**       | ❌ Hanya single column       | ✅ Bisa reference multiple columns |
| **Modification**       | ✅ Bisa ALTER                | ❌ Harus DROP dan RECREATE         |
| **Gradual rollout**    | ✅ Bisa NOT VALIDATED        | ❌ Tidak ada fitur ini             |
| **Encapsulation**      | ✅ Type + constraint bundled | ❌ Terpisah                        |
| **Komunikasi**         | ✅ Nama yang meaningful      | ⚠️ Tergantung naming               |
| **Container types**    | ⚠️ Limitation untuk ALTER    | ✅ No special limitation           |
| **Automatic downcast** | ⚠️ Ya, ke base type          | ❌ N/A                             |

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

-- ⚠️ PENTING: NOT VALIDATED tidak bulletproof!
-- Tidak bisa "see" rows yang sedang di-insert/update concurrent transactions

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

**⚠️ KOREKSI PENTING: Race Condition Risk**

Dokumentasi resmi memperingatkan:

- `ALTER DOMAIN ADD CONSTRAINT` **tidak bulletproof**
- Command tidak bisa "see" rows yang baru di-insert/update dalam uncommitted transactions
- Jika ada concurrent operations, bisa ada bad data masuk

**Safe Workflow:**

```sql
-- 1. Add constraint dengan NOT VALID
ALTER DOMAIN us_postal_code
    ADD CONSTRAINT extended_format
    CHECK (VALUE ~ '^\d{5}$' OR VALUE ~ '^\d{5}-\d{4}$')
    NOT VALID;

-- 2. COMMIT
COMMIT;

-- 3. TUNGGU semua transactions sebelum commit selesai
-- (Monitor dengan pg_stat_activity)

-- 4. VALIDATE constraint (akan check semua data existing)
ALTER DOMAIN us_postal_code VALIDATE CONSTRAINT extended_format;
```

**Workflow Gradual Migration:**

```
1. ALTER DOMAIN dengan NOT VALIDATED
   ↓
2. COMMIT transaction
   ↓
3. Tunggu old transactions finish
   ↓
4. Data baru: Langsung divalidasi dengan rule baru
   Data lama: Tidak di-touch, masih boleh invalid
   ↓
5. Perlahan migrate/fix data lama
   ↓
6. VALIDATE CONSTRAINT untuk enforce ke semua data
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
-- Data lama: Tidak di-check (dengan caveat race condition)
-- Data baru: Divalidasi
```

### h. Automatic Type Downcast - Behavior Penting!

**⚠️ PENAMBAHAN: Behavior yang sering tidak disadari**

Ketika operator atau function dari underlying type diaplikasikan ke domain value, PostgreSQL **automatically downcast** ke underlying type.

```sql
CREATE DOMAIN posint AS INTEGER CHECK (VALUE > 0);

CREATE TABLE mytable (id posint);
INSERT INTO mytable VALUES (5);

-- Query ini menghasilkan INTEGER, BUKAN posint!
SELECT id - 1 FROM mytable;  -- Hasil: INTEGER (bukan posint)

-- Implikasi: Constraint TIDAK di-check ulang!
SELECT (id - 1) FROM mytable WHERE id = 1;
-- Hasil: 0 (INTEGER) - constraint VALUE > 0 TIDAK di-check!

-- Untuk re-check constraint, harus explicit cast:
SELECT (id - 1)::posint FROM mytable WHERE id = 1;
-- ERROR: value for domain posint violates check constraint
```

**Mengapa ini penting?**

```sql
CREATE DOMAIN email AS TEXT CHECK (VALUE LIKE '%@%');

CREATE TABLE users (
    user_email email
);

-- Substring operation: Downcast ke TEXT
SELECT substring(user_email, 1, 5) FROM users;
-- Hasil: TEXT (bukan email)
-- Constraint email TIDAK berlaku lagi!

-- Ini bisa diassign ke kolom TEXT tanpa validasi
CREATE TEMP TABLE temp AS
    SELECT substring(user_email, 1, 5) AS partial FROM users;
-- partial adalah TEXT, bukan email
```

### i. Container Type Limitations - PENTING!

**⚠️ PENAMBAHAN: Limitation yang harus diketahui**

`ALTER DOMAIN ADD CONSTRAINT`, `VALIDATE CONSTRAINT`, dan `SET NOT NULL` akan **GAGAL** jika domain (atau derived domain) digunakan dalam:

- **Composite type column** (row type)
- **Array column**
- **Range column**

```sql
-- Contoh yang akan GAGAL:
CREATE DOMAIN posint AS INTEGER CHECK (VALUE > 0);

-- Domain dalam composite type
CREATE TYPE address_type AS (
    street TEXT,
    zipcode posint  -- Domain dalam composite
);

CREATE TABLE locations (
    location address_type
);

-- INI AKAN GAGAL:
ALTER DOMAIN posint ADD CONSTRAINT max_val CHECK (VALUE < 1000);
-- ERROR: cannot alter type "posint" because column "locations.location" uses it

-- Domain dalam array
CREATE TABLE numbers (
    values posint[]  -- Array of domain
);

-- INI JUGA AKAN GAGAL:
ALTER DOMAIN posint SET NOT NULL;
-- ERROR: cannot alter type "posint" because it is used in array type
```

**Workaround:**

```sql
-- Jika perlu ALTER domain dalam container type:
-- 1. Backup data
-- 2. Drop tables/columns yang pakai container type
-- 3. ALTER domain
-- 4. Recreate tables/columns
-- 5. Restore data

-- Atau: Gunakan base type + check constraint untuk container types
CREATE TABLE locations (
    street TEXT,
    zipcode INTEGER CHECK (zipcode > 0)  -- Bukan domain jika dalam container
);
```

### j. Keuntungan Domain

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

Level 4: Derived Domains
    CREATE DOMAIN limited_price AS positive_price CHECK (VALUE < 1000)
    ↓ (Layered validation)
```

### Domain dalam Ekosistem PostgreSQL:

```
Base Types (INT, TEXT, etc)
    ↓ wrapped by
Domains (custom types with validation)
    ↓ can be base for
Other Domains (derived domains)
    ↓ used in
Tables & Columns (but NOT container types for ALTER)
    ↓ downcast to base type when
Operations/Functions applied
    ↓ protected by
Check Constraints (for multi-column rules)
    ↓ ensures
Data Integrity
```

### Kapan Menggunakan Apa:

```
Single column, used once
    → Check Constraint inline

Single column, used multiple times, NOT in container types
    → Domain

Single column IN container types (composite, array, range)
    → Base type + Check Constraint (easier to maintain)

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

## 4. Kesimpulan

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

```

```
