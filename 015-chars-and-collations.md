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

**Catatan Naming:** PostgreSQL menerima `UTF8` (canonical) dan `UTF-8` (alias) sebagai nama encoding yang sama.

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
SET client_encoding = 'LATIN1';
INSERT INTO messages VALUES ('こんにちは');  -- ❌ ERROR!
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
SELECT datname, datcollate, datctype FROM pg_database;

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
-- Scenario: Client encoding tidak sama dengan server
SET client_encoding = 'SQL_ASCII';  -- Very limited encoding

-- Try to insert emoji
INSERT INTO messages VALUES ('Hello 😀');
-- ⚠️ Akan error atau data loss jika tidak bisa convert

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

**Collation Providers:**

PostgreSQL mendukung dua provider:

- **ICU (International Components for Unicode)**: Recommended untuk custom collations, lebih portable dan konsisten across platforms
- **libc**: System-dependent, behavior bisa berbeda antar OS

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
-- Membuat collation yang case-insensitive menggunakan ICU
CREATE COLLATION case_insensitive (
    provider = icu,
    locale = 'en-US-u-ks-level1',
    deterministic = false
);

-- Alternatif dengan libc (kurang portable)
CREATE COLLATION case_insensitive_libc (
    provider = libc,
    locale = 'en_US.utf8'
);
```

**Breakdown Parameter ICU:**

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
- Setiap karakter punya posisi unik yang pasti

**Deterministic = FALSE:**

- Diperlukan untuk case-insensitive atau accent-insensitive collations
- Karena 'a' = 'A', maka tidak ada ordering relationship yang jelas antara keduanya
- Untuk karakter yang dianggap "equal" oleh collation, PostgreSQL menggunakan binary representation sebagai tie-breaker
- Order tetap konsisten dalam satu query, tapi tidak dijamin spesifik antara karakter yang equal

```sql
-- Dengan deterministic = false
CREATE COLLATION ci (
    provider = icu,
    locale = 'en-US-u-ks-level1',
    deterministic = false
);

SELECT * FROM (VALUES ('abc'), ('ABC'), ('Abc')) AS t(val)
ORDER BY val COLLATE ci;
-- Karena 'abc', 'ABC', 'Abc' dianggap equal,
-- urutan mereka ditentukan oleh binary representation sebagai tie-breaker
-- Output akan konsisten dalam query yang sama, tapi tidak dijamin urutan tertentu
-- Bisa: abc, ABC, Abc (atau urutan lain tergantung binary values)
```

**Penting:** Deterministic false **required** untuk level1 (case/accent insensitive), karena tidak mungkin menentukan ordering yang konsisten ketika 'a' dan 'A' dianggap equal.

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

#### 2. Generated Column (PostgreSQL 12+)

```sql
-- Generated columns tersedia mulai PostgreSQL 12+
CREATE TABLE users (
    email TEXT,
    email_lower TEXT GENERATED ALWAYS AS (LOWER(email)) STORED
);

-- Index pada generated column
CREATE INDEX idx_email_lower ON users(email_lower);

-- Query pakai generated column
SELECT * FROM users WHERE email_lower = LOWER('John@Example.com');
-- ✅ Index akan digunakan!
```

#### 3. Functional Index

```sql
-- Create index on lowercase email
CREATE INDEX idx_email_lower ON users (LOWER(email));

-- Query yang match index
SELECT * FROM users WHERE LOWER(email) = LOWER('John@Example.com');
-- ✅ Index akan dipakai!
```

#### 4. ILIKE Operator

```sql
-- ILIKE = case-insensitive LIKE
SELECT * FROM users WHERE email ILIKE 'john@example.com';  -- ✅

-- PENTING: ILIKE tidak bisa menggunakan regular B-tree index
-- Untuk membuat ILIKE index-assisted, gunakan trigram index:

-- Install extension
CREATE EXTENSION pg_trgm;

-- Create trigram index
CREATE INDEX idx_email_trgm ON users USING gin(email gin_trgm_ops);

-- Sekarang ILIKE bisa menggunakan index
SELECT * FROM users WHERE email ILIKE '%john%';  -- ✅ Index-assisted

-- Alternatif: Functional index untuk exact match
CREATE INDEX idx_email_lower ON users (LOWER(email));
SELECT * FROM users WHERE LOWER(email) = LOWER('john@example.com');
```

### h. Per-Column Collation dan Encoding

Tidak hanya database, kolom individual juga bisa punya collation sendiri (encoding per-column sangat jarang dan tidak recommended).

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
-- Misalnya 'ñ' di Spanish diurutkan sebagai huruf terpisah setelah 'n'
```

## 3. Hubungan Antar Konsep

### Hirarki Character Handling:

```
Database Level
    ├─ Default Encoding (UTF8)
    └─ Default Collation (en_US.UTF-8)
         ↓
Table Level
    ├─ Inherit atau override database defaults
    └─ Each column dapat specify sendiri
         ↓
Column Level
    ├─ Can have own encoding (sangat jarang, tidak recommended)
    └─ Can have own collation (lebih umum dan berguna)
         ↓
Query Level
    └─ COLLATE keyword untuk override sementara
```

### Encoding vs Collation:

```
Character Set (Encoding)
    "Apa karakter yang LEGAL?"
    ├─ UTF8: Semua Unicode ✅
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
│   ├─ Option 2: Generated column + index ✅ Best performance (PG 12+)
│   ├─ Option 3: Functional index ✅ Good balance
│   └─ Option 4: Custom collation ⚠️ Paling complex, use jika benar-benar perlu
└─ Untuk pattern match (LIKE)
    ├─ Option 1: ILIKE dengan trigram index ✅ For wildcard searches
    └─ Option 2: LOWER() + LIKE + functional index ✅ More control
```

### Layer Communication:

```
Application Layer
    ↓ (sends query)
Client Encoding (e.g., UTF8)
    ↓ (transmit over network)
PostgreSQL Server
    ↓ (conversion if needed)
Server Encoding (e.g., UTF8)
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
   SELECT collname, collprovider FROM pg_collation
   ORDER BY collname;

   -- Filter untuk specific locale
   SELECT collname FROM pg_collation
   WHERE collname LIKE 'en_%';

   -- Output:
   -- en_AG
   -- en_AU
   -- en_US
   -- en_US.utf8
   -- ...

   -- Lihat ICU collations saja
   SELECT collname FROM pg_collation
   WHERE collprovider = 'i'  -- 'i' for ICU
   ORDER BY collname;
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
   CREATE COLLATION es_ci (
       provider = icu,
       locale = 'es-u-ks-level1',
       deterministic = false
   );

   -- German: Proper ü, ö, ä sorting
   CREATE COLLATION de_ci (
       provider = icu,
       locale = 'de-u-ks-level1',
       deterministic = false
   );

   -- French: Proper é, è, ê sorting
   CREATE COLLATION fr_ci (
       provider = icu,
       locale = 'fr-u-ks-level1',
       deterministic = false
   );

   -- Japanese: Proper hiragana/katakana sorting
   CREATE COLLATION ja_ci (
       provider = icu,
       locale = 'ja-u-ks-level1',
       deterministic = false
   );
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

-- ✅ FIX: Selalu gunakan UTF8
CREATE DATABASE app ENCODING 'UTF8';


-- ❌ KESALAHAN 3: Custom collation untuk semua kolom
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
-- ERROR atau behavior tidak seperti yang diharapkan

-- ✅ FIX:
CREATE COLLATION ci (
    provider = icu,
    locale = 'en-US-u-ks-level1',
    deterministic = false  -- Wajib untuk case-insensitive!
);


-- ❌ KESALAHAN 5: ILIKE tanpa trigram index untuk pattern match
CREATE TABLE products (name TEXT);
-- Insert 1 million rows
SELECT * FROM products WHERE name ILIKE '%search%';
-- Sangat lambat! Sequential scan

-- ✅ FIX: Gunakan trigram index
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_name_trgm ON products USING gin(name gin_trgm_ops);
SELECT * FROM products WHERE name ILIKE '%search%';
-- Sekarang bisa menggunakan index!

-- Atau untuk exact match:
CREATE INDEX idx_name_lower ON products (LOWER(name));
SELECT * FROM products WHERE LOWER(name) = LOWER('search');


-- ❌ KESALAHAN 6: Menggunakan collation untuk wildcard search
CREATE TABLE products (
    name TEXT COLLATE case_insensitive
);
SELECT * FROM products WHERE name LIKE '%phone%';
-- Tetap case-sensitive karena LIKE tidak respect collation!

-- ✅ FIX: Gunakan ILIKE atau LOWER()
SELECT * FROM products WHERE name ILIKE '%phone%';
-- Atau
SELECT * FROM products WHERE LOWER(name) LIKE LOWER('%phone%');
```

### Performance Considerations:

```sql
-- Custom collation bisa impact performance
CREATE TABLE benchmark (
    col1 TEXT COLLATE "en_US.UTF-8",      -- Standard
    col2 TEXT COLLATE case_insensitive    -- Custom ICU
);

-- Comparison dengan custom collation bisa lebih lambat
-- Terutama untuk operation yang heavy (sorting, grouping, joins)

-- Benchmark estimasi (relatif):
-- Standard collation comparison: 1x
-- Custom ICU collation comparison: 1.2-1.5x
-- LOWER() function: 1.1x
-- Functional index with LOWER(): 1.1x + index benefit

-- Recommendation:
-- 1. Untuk queries yang jarang: Custom collation OK
-- 2. Untuk performance-critical queries: Functional index
-- 3. Untuk bulk operations: LOWER() at insert time + regular index
```

### Real-World Scenarios:

**Scenario 1: Email Addresses**

```sql
-- Best approach: Store lowercase, enforce with constraint
CREATE TABLE users (
    email TEXT CHECK (email = LOWER(email))
);

-- Atau dengan trigger untuk auto-lowercase
CREATE OR REPLACE FUNCTION lowercase_email()
RETURNS TRIGGER AS $$
BEGIN
    NEW.email = LOWER(NEW.email);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER ensure_lowercase_email
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION lowercase_email();
```

**Scenario 2: Product SKUs (Case-Insensitive)**

```sql
-- Option A: Custom collation (jika perlu preserve original case)
CREATE TABLE products (
    sku TEXT COLLATE case_insensitive UNIQUE
);
INSERT INTO products VALUES ('ABC-123');  -- Stored as 'ABC-123'
SELECT * FROM products WHERE sku = 'abc-123';  -- ✅ Found!

-- Option B: Functional unique index (lebih performant)
CREATE TABLE products (
    sku TEXT
);
CREATE UNIQUE INDEX idx_sku_lower ON products (LOWER(sku));
INSERT INTO products VALUES ('ABC-123');
SELECT * FROM products WHERE LOWER(sku) = 'abc-123';  -- ✅ Found!

-- Option C: Store normalized (simplest)
CREATE TABLE products (
    sku TEXT CHECK (sku = UPPER(sku)) UNIQUE
);
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
SELECT * FROM articles ORDER BY title_es;  -- Spanish sorting (ñ sorted correctly)
SELECT * FROM articles ORDER BY title_fr;  -- French sorting (accents matter)
SELECT * FROM articles ORDER BY title_de;  -- German sorting (ü, ö, ä correct)
```

**Scenario 4: Username (Case-Insensitive, Preserve Original)**

```sql
-- Best: Generated column for lookups
CREATE TABLE users (
    username TEXT,
    username_lower TEXT GENERATED ALWAYS AS (LOWER(username)) STORED,
    UNIQUE (username_lower)
);

CREATE INDEX idx_username_lower ON users(username_lower);

-- Insert preserves case
INSERT INTO users (username) VALUES ('JohnDoe');

-- Search is case-insensitive and fast
SELECT * FROM users WHERE username_lower = LOWER('johndoe');
-- Returns: JohnDoe (original case preserved)
```

### Analogi:

**Encoding seperti Alphabet:**

- **ASCII** = Alphabet bahasa Inggris saja (A-Z, 0-9, basic punctuation)
- **Latin1** = Alphabet Western Europe (A-Z, é, ñ, ü, dll)
- **UTF8** = Semua alphabet di dunia (Latin, Chinese, Arabic, Emoji, symbols)

**Collation seperti Aturan Sorting di Kamus:**

- **English dictionary**: 'a' sebelum 'b', case-sensitive, A ≠ a
- **Spanish dictionary**: 'ñ' adalah huruf terpisah setelah 'n'
- **German dictionary**: 'ü' punya posisi khusus dalam alphabet
- **Case-insensitive collation**: 'A' dan 'a' dianggap sama dalam comparison

**Client-Server Encoding seperti Translator:**

```
You (speak French) → Translator → Friend (speak English)
Client (Latin1)    → PostgreSQL → Server (UTF8)

Jika translator tidak tahu kata tertentu = encoding mismatch!
Solusi: Kedua pihak berbicara bahasa yang sama (UTF8)
```

**Deterministic vs Non-Deterministic seperti Ranking:**

```
Deterministic = TRUE (Race dengan timing precision)
- Finish time 1st: 10.01s
- Finish time 2nd: 10.02s
- Finish time 3rd: 10.03s
→ Clear, unambiguous ranking

Deterministic = FALSE (Race dengan same time)
- Finish time 1st: 10.00s
- Finish time 2nd: 10.00s (tie!)
- Finish time 3rd: 10.00s (tie!)
→ Need tie-breaker (lane number = binary representation)
```

### Version Compatibility Notes:

```sql
-- Generated Columns: PostgreSQL 12+
CREATE TABLE t (
    val TEXT,
    val_lower TEXT GENERATED ALWAYS AS (LOWER(val)) STORED
);

-- ICU Collations: PostgreSQL 10+
CREATE COLLATION my_ci (provider = icu, locale = 'en-u-ks-level1', deterministic = false);

-- Nondeterministic Collations: PostgreSQL 12+
-- (deterministic = false parameter)

-- pg_trgm extension: Available in contrib (all versions)
CREATE EXTENSION pg_trgm;
```

### Documentation Links:

- **Collation Support**: https://www.postgresql.org/docs/current/collation.html
- **Locale Support**: https://www.postgresql.org/docs/current/locale.html
- **Character Sets**: https://www.postgresql.org/docs/current/multibyte.html
- **ICU Locales**: https://unicode-org.github.io/icu/userguide/locale/
- **pg_trgm**: https://www.postgresql.org/docs/current/pgtrgm.html

## 5. Kesimpulan

Character sets dan collations adalah aspek fundamental dari bagaimana PostgreSQL menyimpan dan membandingkan text data. Dalam kebanyakan kasus, default settings (UTF8 encoding dengan en_US.UTF-8 collation) sudah sangat baik dan tidak perlu diubah.

**Key Points:**

1. **Encoding (Character Set)**: Mendefinisikan karakter apa saja yang legal (UTF8 recommended untuk semua kasus)
2. **Collation**: Mendefinisikan bagaimana karakter dibandingkan dan diurutkan (default case-sensitive)
3. **Server vs Client**: Ada dua encoding yang berbeda, PostgreSQL akan convert jika perlu dan overlap memungkinkan
4. **Custom Collation**: Bisa dibuat dengan ICU provider untuk case/accent-insensitive, tapi pertimbangkan performance impact
5. **Alternatives**: Untuk case-insensitive search, functional index, generated column, atau normalize input sering lebih baik dan lebih performant
6. **Provider Choice**: ICU provider lebih portable dan konsisten across platforms dibanding libc

**Best Practices:**

- ✅ Gunakan UTF8 encoding (support semua Unicode characters)
- ✅ Stick dengan default collation kecuali ada alasan kuat (en_US.UTF-8)
- ✅ Match client dan server encoding (keduanya UTF8) untuk menghindari conversion issues
- ✅ Untuk case-insensitive: Prefer functional index atau generated column (PostgreSQL 12+)
- ✅ Custom collation hanya untuk kasus spesifik yang tidak bisa diselesaikan cara lain
- ✅ Gunakan ICU provider untuk custom collations (lebih portable)
- ✅ Untuk ILIKE pattern matching, gunakan trigram index (pg_trgm extension)
- ✅ Test performance impact sebelum implement custom collation di production
- ✅ Document collation choices untuk team members

**Kapan Mengubah Defaults:**

- **Multi-language app** → Different collation per language column untuk correct sorting
- **Case-insensitive comparison** → Evaluate: functional index vs generated column vs custom collation
- **Legacy system integration** → Might need specific encoding (tapi migrate ke UTF8 jika memungkinkan)
- **Specific locale requirements** → Custom collation dengan ICU untuk aturan sorting spesifik

**Performance Hierarchy (fastest to slowest untuk case-insensitive search):**

1. **Normalized input + regular index** (e.g., store lowercase) - Fastest
2. **Generated column + index** (PostgreSQL 12+) - Very fast
3. **Functional index** (e.g., `CREATE INDEX ON t(LOWER(col))`) - Fast
4. **Custom collation** - Slower, tapi preserves original case
5. **ILIKE without index** - Slowest (sequential scan)

**Common Pitfalls to Avoid:**

- ❌ Assuming default collation is case-insensitive
- ❌ Using non-UTF8 encoding untuk new projects
- ❌ Creating custom collation when simpler solutions exist
- ❌ Forgetting `deterministic = false` untuk level1 collations
- ❌ Using LIKE instead of ILIKE untuk case-insensitive pattern matching
- ❌ Not creating appropriate indexes untuk case-insensitive searches
- ❌ Mixing client and server encodings without understanding overlap
- ❌ Applying custom collation ke semua columns (overhead tidak perlu)

**Decision Flow Chart:**

```
Perlu case-insensitive handling?
├─ YES
│   ├─ Need to preserve original case?
│   │   ├─ YES
│   │   │   ├─ PostgreSQL 12+? → Use generated column ✅ BEST
│   │   │   └─ PostgreSQL < 12? → Use functional index or custom collation
│   │   └─ NO → Normalize input (LOWER/UPPER) + constraint ✅ SIMPLEST
│   └─ Pattern matching (LIKE)?
│       ├─ Install pg_trgm extension
│       └─ Create trigram index ✅
└─ NO → Use default collation ✅ DEFAULT

Multi-language support?
├─ YES → Use per-column collations with ICU provider ✅
└─ NO → Use default collation ✅

Performance critical?
├─ YES → Benchmark all options, prefer generated column or functional index ✅
└─ NO → Custom collation acceptable ✅
```

**Key Takeaway:**

UTF8 encoding dengan en_US.UTF-8 collation adalah pilihan yang aman dan powerful untuk mayoritas aplikasi modern. Custom collation adalah tool yang powerful untuk kasus spesifik (terutama multi-language apps), tapi untuk use case umum seperti case-insensitive search, pertimbangkan alternatif seperti functional indexes atau generated columns yang lebih performant dan maintainable.

Pahami perbedaan fundamental antara encoding (apa yang legal) dan collation (bagaimana karakter relate) untuk menghindari "mystery bugs" dengan karakter special. Selalu test dengan real-world data yang include accented characters, emoji, dan multi-byte characters untuk ensure aplikasi handle semua edge cases dengan benar.

**Final Recommendations by Use Case:**

| Use Case                  | Recommendation                                | Why                           |
| ------------------------- | --------------------------------------------- | ----------------------------- |
| New project               | UTF8 + default collation                      | Covers 99% of needs           |
| Email addresses           | Normalize to lowercase + check constraint     | Simplest, fastest             |
| Usernames (preserve case) | Generated column (PG 12+) or functional index | Fast, maintains original      |
| Product SKUs              | Normalize to uppercase + unique constraint    | Simplest, consistent          |
| Multi-language content    | Per-column ICU collations                     | Correct sorting per language  |
| Search functionality      | pg_trgm + GIN index                           | Handles wildcards efficiently |
| Legacy system             | Match legacy encoding, plan UTF8 migration    | Compatibility, future-proof   |

**Migration Path:**

Jika saat ini menggunakan encoding lain atau tidak optimal setup:

```sql
-- 1. Assess current setup
SELECT datname, encoding, datcollate, datctype
FROM pg_database
WHERE datname = current_database();

-- 2. Create new database dengan UTF8
CREATE DATABASE myapp_new
    ENCODING 'UTF8'
    LC_COLLATE 'en_US.UTF-8'
    LC_CTYPE 'en_US.UTF-8';

-- 3. Dump dan restore data
-- pg_dump myapp_old | psql myapp_new

-- 4. Test thoroughly
-- 5. Switch over
```

Dengan memahami character sets dan collations dengan baik, developer dapat membuat aplikasi yang robust, internationalization-ready, dan performant dalam handling text data across berbagai bahasa dan character sets.
