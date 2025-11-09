# 📝 Character Sets dan Collations di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang character sets (encoding) dan collations di PostgreSQL, dua konsep penting yang menentukan bagaimana karakter disimpan dan dibandingkan. Character set mendefinisikan karakter apa saja yang legal/valid, sedangkan collation mendefinisikan aturan bagaimana karakter-karakter tersebut saling berhubungan (misalnya apakah 'a' sama dengan 'A'). Video juga menjelaskan cara membuat custom collation dan kapan sebaiknya menggunakan alternatif lain seperti functional indexes.

## 2. Konsep Utama

### a. Character Set / Encoding

Character set (juga disebut encoding atau charset) adalah aturan yang mendefinisikan bagaimana karakter di layar dikonversi menjadi bytes yang disimpan di disk.

**Konsep Dasar:**

```
Character on screen → Encoding → Bytes on disk
     'A'           →  UTF-8   →  0x41
     '😀'          →  UTF-8   →  0xF0 0x9F 0x98 0x80
```

**UTF-8: The Recommended Choice**

UTF-8 adalah multi-byte encoding (1-4 bytes per character) yang sangat versatile:

```sql
-- Melihat encoding database
\l  -- Atau SELECT datname, encoding FROM pg_database;

-- Output contoh:
-- Name    | Encoding | Collate      | Ctype
-- mydb    | UTF8     | en_US.UTF-8  | en_US.UTF-8
```

**Karakteristik UTF-8:**

- ✅ Support seluruh range Unicode
- ✅ Support accented characters (é, ñ, ü)
- ✅ Support uppercase dan lowercase
- ✅ Support emoji (😀, 🎉, 🔥)
- ✅ Variable width: 1-4 bytes per character
- ✅ Backward compatible dengan ASCII

**Contoh:**

```sql
-- UTF-8 bisa handle semua ini:
CREATE TABLE messages (
    content TEXT
);

INSERT INTO messages VALUES ('Hello World');           -- ASCII
INSERT INTO messages VALUES ('Café résumé');          -- Accented
INSERT INTO messages VALUES ('こんにちは');             -- Japanese
INSERT INTO messages VALUES ('مرحبا');                 -- Arabic
INSERT INTO messages VALUES ('Hello 😀🎉');           -- Emoji
-- Semua berhasil dengan UTF-8!

-- Dengan encoding lain (misalnya Latin1):
-- SET client_encoding = 'LATIN1';
-- INSERT INTO messages VALUES ('こんにちは');  -- ❌ ERROR!
-- ERROR: character with byte sequence 0xe3 0x81 0x93 in encoding "UTF8"
--        has no equivalent in encoding "LATIN1"
```

### b. Collation (Kolasi)

Collation adalah set aturan yang mendefinisikan bagaimana karakter-karakter saling berhubungan dan dibandingkan.

**Collation menentukan:**

- Apakah 'a' = 'A'? (case sensitivity)
- Apakah 'e' = 'é'? (accent sensitivity)
- Urutan sorting: 'a' < 'b' < 'c'?
- Bagaimana karakter spesial diurutkan?

**Melihat Collation Default:**

```sql
-- Dari command line
\l

-- Atau dari SQL
SELECT datname, datcollate FROM pg_database;

-- Typical output: en_US.UTF-8
-- en_US = English, United States locale
-- UTF-8 = Character encoding
```

**Karakteristik en_US.UTF-8 Collation:**

```sql
-- Case sensitive
SELECT 'abc' = 'ABC';  -- FALSE

SELECT 'abc' = 'abc';  -- TRUE

-- Lowercase huruf berbeda dari uppercase
SELECT 'Hello' = 'hello';  -- FALSE

-- Accent sensitive
SELECT 'cafe' = 'café';  -- FALSE
```

### c. Server Encoding vs Client Encoding

Ada dua encoding yang berbeda dalam komunikasi PostgreSQL:

#### 1. Server Encoding

Encoding yang digunakan database untuk menyimpan data di disk.

```sql
-- Melihat server encoding
SHOW server_encoding;
-- Output: UTF8

-- Server encoding ditentukan saat create database
CREATE DATABASE mydb
    ENCODING 'UTF8'
    LC_COLLATE 'en_US.UTF-8'
    LC_CTYPE 'en_US.UTF-8';
```

#### 2. Client Encoding

Encoding yang digunakan client (aplikasi/tool) untuk komunikasi dengan server.

```sql
-- Melihat client encoding
SHOW client_encoding;
-- Output: UTF8

-- Mengubah client encoding
SET client_encoding = 'LATIN1';
```

**Conversion Process:**

```
Client (LATIN1) → PostgreSQL Server → Conversion → Database (UTF8)
     ↓                                                      ↓
  "café"           Attempt to convert               Store as UTF8
```

**Potential Issues:**

```sql
-- Scenario: Client encoding tidak overlap dengan server
SET client_encoding = 'SQL_ASCII';  -- Very limited encoding

-- Try to insert emoji
INSERT INTO messages VALUES ('Hello 😀');
-- ⚠️ Might lose emoji atau error jika tidak bisa convert

-- Best practice: Keep both UTF-8!
SET client_encoding = 'UTF8';  -- Match server encoding
```

**Overlap Problem:**

```
Encoding A: {a, b, c, é, ñ}
Encoding B: {a, b, c, x, y, z}

Overlap: {a, b, c}  ✅ Bisa convert
Not in B: {é, ñ}    ❌ Tidak bisa convert → Error atau data loss
```

### d. Case Sensitivity dengan Collation

Default collation (en_US.UTF-8) adalah case-sensitive.

```sql
-- Comparison dengan explicit collation
SELECT 'abc' = 'ABC' COLLATE "en_US.UTF-8";  -- FALSE
-- Case berbeda → tidak sama

SELECT 'abc' = 'abc' COLLATE "en_US.UTF-8";  -- TRUE
-- Case sama → sama

SELECT 'abc' = 'Abc' COLLATE "en_US.UTF-8";  -- FALSE
-- Bahkan satu huruf capital → tidak sama
```

**Real-world Impact:**

```sql
CREATE TABLE users (
    email TEXT
);

INSERT INTO users VALUES ('john@example.com');

-- Query dengan case berbeda
SELECT * FROM users WHERE email = 'John@example.com';
-- ❌ Tidak ketemu! (case sensitive)

-- Solutions:
-- 1. Downcase saat query
SELECT * FROM users WHERE LOWER(email) = LOWER('John@example.com');  -- ✅

-- 2. Gunakan ILIKE
SELECT * FROM users WHERE email ILIKE 'John@example.com';  -- ✅

-- 3. Custom collation (dijelaskan di bawah)
```

### e. Membuat Custom Collation

PostgreSQL memungkinkan kita membuat collation sendiri dengan aturan custom.

**Syntax:**

```sql
CREATE COLLATION collation_name (
    provider = icu,
    locale = 'locale_string',
    deterministic = false
);
```

**Contoh: Case-Insensitive Collation**

```sql
-- Membuat collation yang case-insensitive
CREATE COLLATION case_insensitive (
    provider = icu,
    locale = 'en-US-u-ks-level1',
    deterministic = false
);
```

**Breakdown Parameter:**

| Parameter       | Value         | Meaning                                               |
| --------------- | ------------- | ----------------------------------------------------- |
| `provider`      | `icu`         | International Components for Unicode                  |
| `locale`        | `en-US`       | English, United States                                |
|                 | `u-ks-level1` | Level 1 = base characters only (ignore case & accent) |
| `deterministic` | `false`       | Allow non-deterministic comparison                    |

**Locale String Breakdown:**

```
en-US-u-ks-level1
│  │  │ │  │
│  │  │ │  └─ level1: Base character comparison only
│  │  │ └──── ks: Key strength (comparison sensitivity)
│  │  └─────── u: Unicode extension
│  └────────── US: United States
└───────────── en: English
```

**Levels of Sensitivity:**

- **Level 1**: Base characters (ignore case, ignore accents)
  - 'a' = 'A' = 'á' = 'Á'
- **Level 2**: Accents matter (ignore case)
  - 'a' = 'A', but 'a' ≠ 'á'
- **Level 3**: Case matters (default)
  - 'a' ≠ 'A'

**Testing Custom Collation:**

```sql
-- Sebelum: Case sensitive
SELECT 'abc' = 'ABC';  -- FALSE

-- Setelah: Dengan custom collation
SELECT 'abc' = 'ABC' COLLATE case_insensitive;  -- TRUE ✅

-- Accent juga diabaikan dengan level1
SELECT 'cafe' = 'café' COLLATE case_insensitive;  -- TRUE ✅
```

**Menggunakan di Kolom:**

```sql
-- Apply collation ke kolom tertentu
CREATE TABLE users (
    email TEXT COLLATE case_insensitive,
    name TEXT  -- Tetap pakai default collation
);

INSERT INTO users VALUES ('John@Example.com', 'John Doe');

-- Sekarang email comparison case-insensitive
SELECT * FROM users WHERE email = 'john@example.com';  -- ✅ Ketemu!
SELECT * FROM users WHERE email = 'JOHN@EXAMPLE.COM';  -- ✅ Ketemu juga!
```

### f. Deterministic vs Non-Deterministic

**Deterministic = TRUE (default):**

- Comparison harus konsisten dan predictable
- 'a' selalu < 'b'
- Order sorting selalu sama

**Deterministic = FALSE:**

- Diperlukan untuk case-insensitive atau accent-insensitive
- Karena 'a' = 'A', maka tidak bisa determine mana yang "lebih kecil"
- Order sorting bisa bervariasi untuk karakter yang "equal"

```sql
-- Dengan deterministic = false
CREATE COLLATION ci (provider = icu, locale = 'en-US-u-ks-level1', deterministic = false);

SELECT * FROM (VALUES ('abc'), ('ABC'), ('Abc')) AS t(val) ORDER BY val COLLATE ci;
-- Output bisa:
-- abc
-- ABC
-- Abc
-- Atau urutan lain (karena semua "equal")

-- Dengan deterministic = true (tidak bisa level1)
-- Harus ada tie-breaker yang jelas
```

### g. Alternatif untuk Case-Insensitive Search

Video menyebutkan bahwa untuk pencarian case-insensitive, ada cara yang lebih baik dari membuat custom collation:

#### 1. Downcase pada Input

```sql
-- Store email dalam lowercase
CREATE TABLE users (
    email TEXT CHECK (email = LOWER(email))
);

-- Atau pakai trigger untuk auto-lowercase
CREATE OR REPLACE FUNCTION lowercase_email()
RETURNS TRIGGER AS $$
BEGIN
    NEW.email = LOWER(NEW.email);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER lowercase_email_trigger
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION lowercase_email();
```

#### 2. Generated Column (akan dibahas di video lain)

```sql
CREATE TABLE users (
    email TEXT,
    email_lower TEXT GENERATED ALWAYS AS (LOWER(email)) STORED
);

-- Index pada generated column
CREATE INDEX idx_email_lower ON users(email_lower);

-- Query pakai generated column
SELECT * FROM users WHERE email_lower = LOWER('John@Example.com');
```

#### 3. Functional Index (akan dibahas di video lain)

```sql
-- Create index on lowercase email
CREATE INDEX idx_email_lower ON users (LOWER(email));

-- Query yang match index
SELECT * FROM users WHERE LOWER(email) = LOWER('John@Example.com');
-- Index akan dipakai! ✅
```

#### 4. ILIKE Operator

```sql
-- ILIKE = case-insensitive LIKE
SELECT * FROM users WHERE email ILIKE 'john@example.com';  -- ✅

-- Tapi ILIKE tidak selalu index-assisted
-- Untuk exact match, cara 1-3 lebih baik
```

### h. Per-Column Collation dan Encoding

Tidak hanya database, kolom individual juga bisa punya collation/encoding sendiri.

```sql
-- Database dengan default en_US.UTF-8
CREATE DATABASE mydb
    ENCODING 'UTF8'
    LC_COLLATE 'en_US.UTF-8';

-- Table dengan mixed collations
CREATE TABLE products (
    name_en TEXT COLLATE "en_US.UTF-8",      -- English names (case-sensitive)
    name_fr TEXT COLLATE "fr_FR.UTF-8",      -- French names
    name_de TEXT COLLATE "de_DE.UTF-8",      -- German names
    sku TEXT COLLATE case_insensitive        -- SKU codes (case-insensitive)
);

-- Different collation untuk different use cases
```

**Use Case:**

```sql
-- Multi-language application
CREATE TABLE articles (
    title_en TEXT COLLATE "en_US.UTF-8",
    title_es TEXT COLLATE "es_ES.UTF-8",     -- Spanish (ñ sorting)
    title_fr TEXT COLLATE "fr_FR.UTF-8"      -- French (é, è, ê sorting)
);

-- Setiap bahasa punya aturan sorting berbeda!
-- Misalnya 'ñ' di Spanish diurutkan berbeda dengan 'n' biasa
```

## 3. Hubungan Antar Konsep

### Hirarki Character Handling:

```
Database Level
    ├─ Default Encoding (UTF-8)
    └─ Default Collation (en_US.UTF-8)
         ↓
Table Level
    ├─ Inherit atau override database defaults
    └─ Each column dapat specify sendiri
         ↓
Column Level
    ├─ Can have own encoding (jarang)
    └─ Can have own collation (lebih umum)
         ↓
Query Level
    └─ COLLATE keyword untuk override sementara
```

### Encoding vs Collation:

```
Character Set (Encoding)
    "Apa karakter yang LEGAL?"
    ├─ UTF-8: Semua Unicode ✅
    ├─ Latin1: Western European only
    └─ ASCII: Basic English only
         ↓
         Defines valid characters
         ↓
Collation
    "Bagaimana karakter RELATE?"
    ├─ Case sensitivity: 'a' vs 'A'
    ├─ Accent sensitivity: 'e' vs 'é'
    └─ Sorting order: 'a' < 'b' < 'c'
```

### Decision Tree untuk Case-Insensitive:

```
Butuh case-insensitive comparison?
├─ Untuk exact match (=)
│   ├─ Option 1: LOWER() pada input ✅ Simplest
│   ├─ Option 2: Generated column + index ✅ Best performance
│   ├─ Option 3: Functional index ✅ Good balance
│   └─ Option 4: Custom collation ⚠️ Paling complex
└─ Untuk pattern match (LIKE)
    ├─ Option 1: ILIKE ✅ Simple
    └─ Option 2: LOWER() + LIKE ✅ More control
```

### Layer Communication:

```
Application Layer
    ↓ (sends query)
Client Encoding (e.g., UTF-8)
    ↓ (transmit over network)
PostgreSQL Server
    ↓ (conversion if needed)
Server Encoding (e.g., UTF-8)
    ↓ (apply collation rules)
Collation (e.g., en_US.UTF-8)
    ↓ (store/retrieve)
Disk Storage (bytes)
```

## 4. Catatan Tambahan / Insight

### Tips Praktis:

1. **Stick to Defaults (Hampir Selalu):**

   ```sql
   -- Ini sudah sangat baik untuk 99% kasus
   CREATE DATABASE myapp
       ENCODING 'UTF8'
       LC_COLLATE 'en_US.UTF-8'
       LC_CTYPE 'en_US.UTF-8';

   -- Jangan ubah kecuali ada alasan kuat!
   ```

2. **Listing Available Collations:**

   ```sql
   -- Lihat semua collations yang tersedia
   SELECT collname FROM pg_collation ORDER BY collname;

   -- Filter untuk specific locale
   SELECT collname FROM pg_collation
   WHERE collname LIKE 'en_%';

   -- Output:
   -- en_AG
   -- en_AU
   -- en_US
   -- en_US.utf8
   -- ...
   ```

3. **Testing Collation Behavior:**

   ```sql
   -- Test case sensitivity
   SELECT
       'abc' = 'ABC' AS default_collation,
       'abc' = 'ABC' COLLATE case_insensitive AS custom_collation;
   -- Result: false | true

   -- Test accent sensitivity
   SELECT
       'cafe' = 'café' COLLATE "en_US.UTF-8" AS default_col,
       'cafe' = 'café' COLLATE case_insensitive AS custom_col;
   -- Result: false | true (if level1)
   ```

4. **Common Collations untuk Different Languages:**

   ```sql
   -- Spanish: Proper ñ sorting
   CREATE COLLATION es_ci (provider = icu, locale = 'es-u-ks-level1', deterministic = false);

   -- German: Proper ü, ö, ä sorting
   CREATE COLLATION de_ci (provider = icu, locale = 'de-u-ks-level1', deterministic = false);

   -- French: Proper é, è, ê sorting
   CREATE COLLATION fr_ci (provider = icu, locale = 'fr-u-ks-level1', deterministic = false);

   -- Japanese: Proper hiragana/katakana sorting
   CREATE COLLATION ja_ci (provider = icu, locale = 'ja-u-ks-level1', deterministic = false);
   ```

5. **Checking Column Collation:**
   ```sql
   -- Lihat collation dari kolom
   SELECT
       table_name,
       column_name,
       collation_name
   FROM information_schema.columns
   WHERE table_schema = 'public'
     AND collation_name IS NOT NULL;
   ```

### Kesalahan Umum:

```sql
-- ❌ KESALAHAN 1: Mengira case-insensitive secara default
CREATE TABLE users (email TEXT);
INSERT INTO users VALUES ('john@example.com');
SELECT * FROM users WHERE email = 'John@example.com';
-- Tidak ketemu! Default adalah case-sensitive

-- ✅ FIX:
SELECT * FROM users WHERE LOWER(email) = LOWER('John@example.com');


-- ❌ KESALAHAN 2: Encoding mismatch
CREATE DATABASE app ENCODING 'LATIN1';
INSERT INTO table VALUES ('😀');  -- ERROR!
-- Latin1 tidak support emoji

-- ✅ FIX: Selalu gunakan UTF-8
CREATE DATABASE app ENCODING 'UTF8';


-- ❌ KESALAHAN 3: Custom collation untuk semua
CREATE COLLATION my_collation (...);
-- Apply ke SEMUA kolom
-- Masalah: Overhead performance untuk kolom yang tidak perlu

-- ✅ FIX: Apply hanya ke kolom yang butuh
CREATE TABLE users (
    id SERIAL,                              -- Default collation OK
    email TEXT COLLATE case_insensitive,    -- Custom untuk email
    name TEXT                               -- Default collation OK
);


-- ❌ KESALAHAN 4: Lupa deterministic = false
CREATE COLLATION ci (
    provider = icu,
    locale = 'en-US-u-ks-level1'
    -- Lupa: deterministic = false
);
-- Mungkin error atau behavior tidak seperti yang diharapkan

-- ✅ FIX:
CREATE COLLATION ci (
    provider = icu,
    locale = 'en-US-u-ks-level1',
    deterministic = false  -- Penting untuk case-insensitive!
);


-- ❌ KESALAHAN 5: ILIKE tanpa index
CREATE TABLE products (name TEXT);
-- Insert 1 million rows
SELECT * FROM products WHERE name ILIKE '%search%';
-- Sangat lambat! Sequential scan

-- ✅ FIX: Gunakan functional index atau generated column
CREATE INDEX idx_name_lower ON products (LOWER(name));
SELECT * FROM products WHERE LOWER(name) LIKE LOWER('%search%');
```

### Performance Considerations:

```sql
-- Custom collation bisa impact performance
CREATE TABLE benchmark (
    col1 TEXT COLLATE "en_US.UTF-8",      -- Standard
    col2 TEXT COLLATE case_insensitive    -- Custom
);

-- Comparison dengan custom collation bisa lebih lambat
-- Terutama untuk operation yang heavy (sorting, grouping)

-- Benchmark (example numbers):
-- Standard collation sort: 100ms
-- Custom collation sort: 150ms
-- Functional index with LOWER(): 110ms

-- Recommendation:
-- Untuk performance-critical queries, prefer functional index
CREATE INDEX idx_col_lower ON table (LOWER(column));
```

### Real-World Scenarios:

**Scenario 1: Email Addresses**

```sql
-- Best approach: Store lowercase, enforce with check
CREATE TABLE users (
    email TEXT CHECK (email = LOWER(email))
);

-- Or with trigger
CREATE TRIGGER ensure_lowercase_email
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION lowercase_email();
```

**Scenario 2: Product SKUs (Case-Insensitive)**

```sql
-- Option A: Custom collation
CREATE TABLE products (
    sku TEXT COLLATE case_insensitive UNIQUE
);

-- Option B: Functional unique index (lebih flexible)
CREATE TABLE products (
    sku TEXT
);
CREATE UNIQUE INDEX idx_sku_lower ON products (LOWER(sku));
```

**Scenario 3: Multi-Language App**

```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title_en TEXT COLLATE "en_US.UTF-8",
    title_es TEXT COLLATE "es_ES.UTF-8",
    title_fr TEXT COLLATE "fr_FR.UTF-8",
    title_de TEXT COLLATE "de_DE.UTF-8"
);

-- Each language column sorts correctly per its rules
SELECT * FROM articles ORDER BY title_es;  -- Spanish sorting
SELECT * FROM articles ORDER BY title_fr;  -- French sorting
```

### Analogi:

**Encoding seperti Alphabet:**

- **ASCII** = Alphabet bahasa Inggris saja (A-Z, 0-9)
- **Latin1** = Alphabet Western Europe (A-Z, é, ñ, ü)
- **UTF-8** = Semua alphabet di dunia (Latin, Chinese, Arabic, Emoji)

**Collation seperti Aturan Sorting di Kamus:**

- **English dictionary**: 'a' sebelum 'b', case-sensitive
- **Spanish dictionary**: 'ñ' diurutkan setelah 'n', beda dari 'n' biasa
- **German dictionary**: 'ü' punya posisi khusus
- **Case-insensitive**: 'A' dan 'a' dianggap sama, diurutkan bersama

**Client-Server Encoding seperti Translator:**

```
You (speak French) → Translator → Friend (speak English)
Client (Latin1)    → PostgreSQL → Server (UTF-8)

Jika translator tidak tahu kata tertentu = encoding mismatch!
```

### Documentation Links:

```sql
-- ICU Locale documentation
-- https://www.postgresql.org/docs/current/collation.html

-- Collation support
-- https://www.postgresql.org/docs/current/locale.html

-- Character set support
-- https://www.postgresql.org/docs/current/multibyte.html
```

## 5. Kesimpulan

Character sets dan collations adalah aspek fundamental dari bagaimana PostgreSQL menyimpan dan membandingkan text data. Dalam kebanyakan kasus, default settings (UTF-8 encoding dengan en_US.UTF-8 collation) sudah sangat baik dan tidak perlu diubah.

**Key Points:**

1. **Encoding (Character Set)**: Mendefinisikan karakter apa saja yang legal (UTF-8 recommended)
2. **Collation**: Mendefinisikan bagaimana karakter dibandingkan dan diurutkan
3. **Server vs Client**: Ada dua encoding yang berbeda, PostgreSQL akan convert jika perlu
4. **Custom Collation**: Bisa dibuat dengan ICU provider untuk case/accent-insensitive
5. **Alternatives**: Untuk case-insensitive search, functional index atau generated column sering lebih baik

**Best Practices:**

- ✅ Gunakan UTF-8 encoding (support semua Unicode)
- ✅ Stick dengan default collation kecuali ada alasan kuat
- ✅ Untuk case-insensitive: Prefer functional index atau lowercase input
- ✅ Custom collation hanya untuk kasus spesifik yang tidak bisa diselesaikan cara lain
- ✅ Match client dan server encoding (keduanya UTF-8)

**Kapan Mengubah Defaults:**

- Multi-language app → Different collation per language column
- Case-insensitive comparison → Custom collation atau functional index
- Legacy system → Might need specific encoding

**Key takeaway:** UTF-8 encoding dengan en_US.UTF-8 collation adalah pilihan yang aman dan powerful untuk mayoritas aplikasi. Custom collation adalah tool yang powerful, tapi untuk case-insensitive search, pertimbangkan alternatif seperti functional indexes yang lebih performant dan maintainable. Pahami perbedaan encoding dan collation untuk menghindari "mystery bugs" dengan karakter special.
