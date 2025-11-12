# 📝 Arrays di PostgreSQL

## 1. Ringkasan Singkat

Materi ini membahas tentang **arrays** sebagai tipe data standalone di PostgreSQL (berbeda dengan arrays di dalam JSON). PostgreSQL memiliki dukungan yang sangat robust untuk arrays dengan banyak fungsi dan operator untuk manipulasi data. Video ini menjelaskan cara mendeklarasikan array columns, kapan sebaiknya menggunakan arrays versus tabel terpisah, cara insert data array, dan preview beberapa operator powerful untuk bekerja dengan arrays seperti indexing, slicing, filtering, dan `unnest`.

## 2. Konsep Utama

### a. Apa itu Array Data Type di PostgreSQL?

- **Arrays** adalah tipe data standalone di PostgreSQL (bukan array di dalam JSON)
- Setiap tipe data PostgreSQL bisa dijadikan array
- PostgreSQL menyediakan banyak **fungsi dan operator** untuk bekerja dengan arrays
- Sangat powerful dan flexible untuk use case tertentu

**Perbedaan dengan JSON arrays:**

```
JSON Array:                    Native Array:
└─ Part of JSON document      └─ Standalone column type
└─ Zero-based indexing        └─ One-based indexing ⚠️
└─ Flexible types             └─ Typed (semua element sama type)
```

### b. Cara Mendeklarasikan Array Columns

PostgreSQL menyediakan dua sintaks untuk mendeklarasikan array columns:

#### **Sintaks 1: Menggunakan Square Brackets `[]`**

```sql
CREATE TABLE array_example (
    id BIGINT PRIMARY KEY,
    integer_array INTEGER[],      -- Array of integers
    text_array TEXT[],            -- Array of text
    boolean_array BOOLEAN[]       -- Array of booleans
);
```

#### **Sintaks 2: Menggunakan Keyword `ARRAY`**

```sql
CREATE TABLE array_example (
    id BIGINT PRIMARY KEY,
    integer_array ARRAY,          -- Sama dengan INTEGER[]
    text_array ARRAY,             -- Sama dengan TEXT[]
    boolean_array ARRAY           -- Sama dengan BOOLEAN[]
);
```

**Kedua sintaks ini equivalent dan sama-sama valid.**

#### **Nested Arrays (Multidimensional)**

```sql
CREATE TABLE array_example (
    id BIGINT PRIMARY KEY,
    nested_array INTEGER[][]      -- Array of arrays (2D)
);

-- Atau dengan sintaks ARRAY
nested_array ARRAY ARRAY          -- Nested array
```

### c. Cara Insert Data ke Array Columns

PostgreSQL menyediakan dua format untuk insert array data:

#### **Format 1: Constructor Syntax (Verbose, Lebih Jelas)**

```sql
INSERT INTO array_example (integer_array, text_array, boolean_array)
VALUES (
    ARRAY[1, 2, 3],                    -- Integer array
    ARRAY['rose', 'daisy', 'poppy'],   -- Text array
    ARRAY[true, false, true]           -- Boolean array
);
```

#### **Format 2: Curly Brace Syntax (Compact)**

```sql
INSERT INTO array_example (integer_array, text_array, boolean_array)
VALUES (
    '{1, 2, 3}',                       -- Integer array
    '{rose, daisy, poppy}',            -- Text array
    '{true, false, true}'              -- Boolean array
);
```

#### **Nested Arrays**

```sql
-- Nested array dengan curly brace syntax
INSERT INTO array_example (nested_array)
VALUES (
    '{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}}'
);

-- Hasilnya adalah array 3x3:
-- [
--   [1, 2, 3],
--   [4, 5, 6],
--   [7, 8, 9]
-- ]
```

**Output format:**

- PostgreSQL selalu mengembalikan arrays dalam **curly brace format**
- Format ini adalah standard output representation

```sql
SELECT * FROM array_example;
-- Output:
-- integer_array: {1,2,3}
-- text_array: {rose,daisy,poppy}
-- nested_array: {{1,2,3},{4,5,6},{7,8,9}}
```

### d. Operator dan Fungsi untuk Arrays

#### **1. Indexing: Mengakses Element Tunggal**

**⚠️ PENTING: PostgreSQL menggunakan ONE-BASED INDEXING untuk arrays!**

```sql
SELECT text_array[1] FROM array_example;
-- Output: 'rose' (element pertama)

-- ❌ TIDAK ADA index [0]
SELECT text_array[0] FROM array_example;
-- Output: NULL (tidak ada element ke-0)

SELECT text_array[3] FROM array_example;
-- Output: 'poppy' (element ketiga)
```

**Perbandingan dengan JSON arrays:**

```sql
-- Native arrays: ONE-BASED (dimulai dari 1)
SELECT ARRAY['a', 'b', 'c'][1];  -- Output: 'a'

-- JSON arrays: ZERO-BASED (dimulai dari 0)
SELECT '["a", "b", "c"]'::JSONB -> 0;  -- Output: "a"
```

#### **2. Slicing: Mengambil Subset Array**

```sql
-- Closed slice (from 1 to 3)
SELECT text_array[1:3] FROM array_example;
-- Output: {rose,daisy,poppy}

-- Open-ended slice (from start to 3)
SELECT text_array[:3] FROM array_example;
-- Output: {rose,daisy,poppy}

-- Open-ended slice (from 3 to end)
SELECT text_array[3:] FROM array_example;
-- Output: {poppy}

-- Slice dengan range lebih besar dari ukuran array (aman)
SELECT text_array[1:10] FROM array_example;
-- Output: {rose,daisy,poppy} (tidak error)
```

#### **3. Array Contains Operator: `@>`**

Operator untuk memeriksa apakah array **mengandung** element atau subset tertentu:

```sql
-- Cek apakah array mengandung 'poppy'
SELECT id, text_array
FROM array_example
WHERE text_array @> ARRAY['poppy'];
-- Returns rows yang text_array-nya mengandung 'poppy'

-- Shorthand (untuk single element)
SELECT id, text_array
FROM array_example
WHERE text_array @> '{poppy}';
-- Sama dengan query di atas

-- Cek multiple elements
SELECT id, text_array
FROM array_example
WHERE text_array @> ARRAY['rose', 'poppy'];
-- Returns rows yang mengandung KEDUA element
```

**Use case:**

- Filter berdasarkan tags
- Cari records dengan specific values in array
- Check membership

#### **4. UNNEST: Convert Array ke Rows**

`UNNEST` adalah fungsi yang sangat powerful yang mengubah array menjadi **result set** (rows):

```sql
-- Simple unnest
SELECT UNNEST(text_array) AS flower
FROM array_example;

-- Output:
-- flower
-- ------
-- rose
-- daisy
-- poppy
```

**Unnest dengan kolom lain:**

```sql
SELECT
    id,
    UNNEST(text_array) AS flower
FROM array_example;

-- Output (kolom non-array akan di-copy untuk setiap row):
-- id | flower
-- ---|-------
-- 1  | rose
-- 1  | daisy
-- 1  | poppy
```

**Unnest dalam CTE untuk query lebih kompleks:**

```sql
-- Menggunakan CTE untuk filtering
WITH flowers AS (
    SELECT
        id,
        UNNEST(text_array) AS flower
    FROM array_example
)
SELECT *
FROM flowers
WHERE flower = 'poppy';

-- Output hanya rows dengan flower = 'poppy'
```

**Kegunaan UNNEST:**

- Convert "array land" kembali ke "SQL land"
- Enable standard SQL operations pada array elements
- Join dengan tables lain
- Aggregations pada array elements

### e. Kapan Menggunakan Arrays vs Tabel Terpisah?

Ini adalah pertanyaan design yang penting dan **case-by-case basis**.

#### **✅ Use Case yang Cocok untuk Arrays**

**1. Sensor Readings atau Time-Series Data**

```sql
CREATE TABLE sensor_data (
    sensor_id BIGINT,
    timestamp TIMESTAMPTZ,
    readings INTEGER[]  -- Array of readings dalam 1 menit
);

-- Contoh: Sensor mengambil 60 readings per menit
INSERT INTO sensor_data VALUES
    (1, NOW(), ARRAY[20, 21, 20, 22, 21, ...]);  -- 60 values
```

**Alasan cocok:**

- Readings selalu diakses **sebagai satu kesatuan**
- Tidak perlu query individual readings
- Compact storage
- Retrieval lebih cepat (satu row vs 60 rows)

**2. Tags atau Labels**

```sql
CREATE TABLE articles (
    id BIGINT PRIMARY KEY,
    title TEXT,
    tags TEXT[]  -- Array of tag names atau tag IDs
);

INSERT INTO articles VALUES
    (1, 'PostgreSQL Tutorial', ARRAY['database', 'sql', 'tutorial']);
```

**Alasan cocok:**

- Tags sering di-query bersama-sama
- Tidak perlu intermediate junction table
- Simplifikasi schema

**⚠️ Trade-off:**

- **Kehilangan referential integrity** jika menggunakan tag IDs
- Jika tag dihapus dari master table, tidak otomatis cleanup dari arrays
- Lebih sulit maintain consistency

#### **❌ Kapan Sebaiknya TIDAK Menggunakan Arrays**

**1. Ketika butuh referential integrity strict**

```sql
-- ❌ Problematic dengan arrays
CREATE TABLE posts (
    id BIGINT PRIMARY KEY,
    category_ids INTEGER[]  -- Tidak ada FK constraint
);

-- ✅ Lebih baik dengan junction table
CREATE TABLE posts (
    id BIGINT PRIMARY KEY
);

CREATE TABLE post_categories (
    post_id BIGINT REFERENCES posts(id),
    category_id BIGINT REFERENCES categories(id),
    PRIMARY KEY (post_id, category_id)
);
```

**2. Ketika perlu query/update individual elements sering**

- Jika sering perlu filter/sort/group by individual elements
- Jika individual elements punya attributes sendiri
- Lebih baik normalized ke tabel terpisah

**3. Ketika array bisa tumbuh sangat besar**

- Performance penalty untuk arrays besar
- Consider normalized structure

#### **Decision Matrix**

```
Pertimbangkan Arrays jika:
✅ Data selalu diakses bersama-sama
✅ Tidak butuh strict referential integrity
✅ Array size relatif kecil dan bounded
✅ Schema lebih simple lebih penting
✅ Performance gain dari denormalization

Hindari Arrays jika:
❌ Perlu strict referential integrity
❌ Sering query individual elements
❌ Array bisa tumbuh unbounded
❌ Elements punya attributes kompleks
❌ Perlu normalized structure untuk reporting
```

### f. Karakteristik Penting Arrays di PostgreSQL

**1. Type Safety**

- Semua elements dalam array **harus same type**
- PostgreSQL enforce type checking

```sql
-- ❌ Error: tidak bisa mix types
INSERT INTO array_example (integer_array)
VALUES (ARRAY[1, 'two', 3]);  -- Error!

-- ✅ Correct: semua integers
INSERT INTO array_example (integer_array)
VALUES (ARRAY[1, 2, 3]);
```

**2. One-Based Indexing (Unusual!)**

- Berbeda dengan kebanyakan programming languages
- JSON arrays tetap zero-based
- Hati-hati saat migrasi dari systems lain

**3. NULL Handling**

```sql
-- Array bisa contain NULL values
SELECT ARRAY[1, NULL, 3];
-- Output: {1,NULL,3}

-- Array itu sendiri bisa NULL
SELECT NULL::INTEGER[];
-- Output: NULL
```

**4. Size Limits**

- Tidak ada hard limit untuk array size
- Tapi performance menurun untuk arrays sangat besar
- Best practice: keep arrays reasonably sized

## 3. Hubungan Antar Konsep

Arrays di PostgreSQL melengkapi ecosystem tipe data yang sudah ada:

```
Data Storage Strategies
         |
    ┌────┴─────┐
    |          |
Structured  Semi-structured
    |          |
    |      ┌───┴────┐
    |      |        |
Discrete  JSON    Arrays
Columns           |
    |          Multiple
    |          values,
Foreign    same type
Keys       |
    |      └─── Trade-off:
Junction       Simplicity
Tables         vs
    |          Normalization
    └──────────┘
```

**Alur keputusan storage:**

1. **Start with normalized structure** → Foreign keys + junction tables
2. **Identify repeated patterns** → Candidate untuk arrays atau JSON
3. **Evaluate access patterns:**
   - Sering akses bersama? → Arrays
   - Perlu flexibility? → JSON
   - Butuh referential integrity? → Stay normalized
4. **Consider trade-offs** dan pilih yang best fit

**Arrays vs JSON vs Normalized:**

| Aspect                    | Arrays       | JSON         | Normalized Tables |
| ------------------------- | ------------ | ------------ | ----------------- |
| **Type safety**           | ✅ Strong    | ❌ Weak      | ✅ Strong         |
| **Referential integrity** | ❌ No        | ❌ No        | ✅ Yes            |
| **Query flexibility**     | 🟡 Good      | 🟡 Good      | ✅ Excellent      |
| **Schema simplicity**     | ✅ Simple    | ✅ Simple    | ❌ Complex        |
| **Performance (bulk)**    | ✅ Fast      | ✅ Fast      | 🟡 Slower         |
| **Indexing**              | ✅ Supported | ✅ Supported | ✅ Native         |

**Kombinasi strategies:**

```sql
-- Hybrid approach: best of both worlds
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name TEXT,

    -- Normalized: strict integrity
    category_id BIGINT REFERENCES categories(id),

    -- Arrays: simple repeated values
    image_urls TEXT[],

    -- JSON: flexible attributes
    specifications JSONB
);
```

**Operator ecosystem:**

```
Array Operations:
    |
    ├─ Indexing ([n])        → Access single element
    ├─ Slicing ([m:n])       → Get subset
    ├─ Contains (@>)         → Membership check
    ├─ UNNEST()              → Convert to rows
    ├─ array_length()        → Get size
    ├─ array_append()        → Add element
    ├─ array_cat()           → Concatenate
    └─ ... many more         → (akan dibahas di querying section)
```

## 4. Kesimpulan

- PostgreSQL mendukung **arrays sebagai tipe data standalone** yang sangat powerful
- Arrays bisa dideklarasikan dengan `TYPE[]` atau `ARRAY` keyword
- **Two formats untuk insert**: `ARRAY[...]` constructor atau `{...}` curly brace syntax
- **⚠️ PostgreSQL arrays menggunakan ONE-BASED INDEXING** (dimulai dari 1, bukan 0)
- **Key operators dan functions:**
  - **Indexing** `[n]` → akses element
  - **Slicing** `[m:n]` → ambil subset
  - **Contains** `@>` → check membership
  - **UNNEST** → convert array ke rows (sangat powerful!)
- **Kapan menggunakan arrays:**
  - ✅ Data selalu diakses bersama-sama
  - ✅ Schema simplicity lebih penting
  - ✅ Array size bounded dan reasonable
  - ❌ Hindari jika butuh strict referential integrity
  - ❌ Hindari jika sering query individual elements
- **Trade-offs:**
  - 👍 Simplifikasi schema (no junction tables)
  - 👍 Performance untuk bulk access
  - 👎 Kehilangan referential integrity
  - 👎 Lebih sulit maintain consistency
- Arrays sangat berguna untuk **sensor data, tags, lists of values** yang naturally grouped together
- PostgreSQL menyediakan **robust functions dan operators** untuk manipulasi arrays (akan dibahas lebih detail di section querying dan indexing)

**Template use case yang cocok:**

```sql
-- Sensor/IoT data
CREATE TABLE sensor_readings (
    sensor_id BIGINT,
    timestamp TIMESTAMPTZ,
    temperature_readings NUMERIC[],  -- Array of readings
    PRIMARY KEY (sensor_id, timestamp)
);

-- Tags/Labels
CREATE TABLE blog_posts (
    id BIGINT PRIMARY KEY,
    title TEXT,
    content TEXT,
    tags TEXT[],                     -- Array of tag names
    created_at TIMESTAMPTZ
);

-- Multiple URLs/Paths
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name TEXT,
    image_urls TEXT[],               -- Array of image URLs
    document_paths TEXT[]            -- Array of file paths
);
```
