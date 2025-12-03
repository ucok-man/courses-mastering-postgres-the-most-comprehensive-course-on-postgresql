# 📝 Composite Types di PostgreSQL: Menggabungkan Multiple Types dalam Satu Unit

## 1. Ringkasan Singkat

Video ini membahas **composite types** di PostgreSQL, yaitu tipe data yang menggabungkan beberapa tipe data PostgreSQL menjadi **satu unit yang tidak terpisahkan**. Meskipun fitur ini menarik dan powerful, composite types **jarang digunakan** dalam praktik sehari-hari. Pembahasan mencakup cara membuat composite type, cara instantiate-nya, cara menyimpannya di tabel, dan cara mengakses field individual dari composite type. Video juga menjelaskan kapan sebaiknya menggunakan composite types dan alternatif apa yang lebih umum dipakai.

---

## 2. Konsep Utama

### a. Apa itu Composite Types?

**Definisi:**
Composite type adalah tipe data kustom yang **menggabungkan beberapa tipe data PostgreSQL** menjadi **satu unit diskrit** yang tidak terpisahkan.

**Karakteristik:**

- Membungkus multiple types menjadi **satu compressed unit of information**
- Mirip dengan **struct** di bahasa pemrograman lain
- **Inseparable unit**: semua field diperlakukan sebagai satu kesatuan
- **Tidak bisa memiliki constraints** (seperti `NOT NULL`) di level type definition

**Kapan composite types berguna:**

- Ketika ada **group of related fields** yang selalu muncul bersama
- Digunakan oleh **extensions** seperti PostGIS (contoh: `address` type dengan 18 fields)
- Untuk membuat **reusable data structures** di database

**⚠️ DISCLAIMER PENTING:**
Composite types **jarang digunakan** dalam praktik! Biasanya alternatif berikut lebih baik:

- **Discrete columns** di table
- **JSON/JSONB column** untuk semi-structured data
- **Separate table** dengan relasi (normalization)

---

### b. Cara Membuat Composite Type

**Sintaks:**

```sql
CREATE TYPE type_name AS (
  field1 type1,
  field2 type2,
  field3 type3
);
```

**⚠️ Keyword `AS` sangat penting!**
Tanpa `AS`, PostgreSQL akan mengira Anda membuat tipe data berbeda (enum atau domain).

**Contoh: Address Type**

```sql
CREATE TYPE address AS (
  number INT,
  street TEXT,
  city TEXT,
  state TEXT,
  zip TEXT
);
```

**Penjelasan:**

- `address` adalah nama composite type
- Memiliki 5 fields: number (int), street (text), city (text), state (text), zip (text)
- Syntax mirip dengan **table constructor**, tapi ini bukan table!

**Keterbatasan:**

```sql
-- ❌ TIDAK BISA: constraints di type definition
CREATE TYPE address AS (
  number INT NOT NULL,  -- ERROR!
  street TEXT
);

-- ✅ BISA: constraints di table level (nanti)
CREATE TABLE addresses (
  id SERIAL PRIMARY KEY,
  addr address NOT NULL  -- constraint di sini OK
);
```

---

### c. Cara Instantiate (Membuat Instance) Composite Type

Ada beberapa cara untuk membuat instance dari composite type:

**Metode 1: Menggunakan `ROW()` constructor**

```sql
SELECT ROW(123, 'Main', 'Anytown', 'ST', '12345')::address;
-- Output: (123,Main,Anytown,ST,12345)

-- Cek tipe data
SELECT pg_typeof(
  ROW(123, 'Main', 'Anytown', 'ST', '12345')::address
);
-- Output: address
```

**Metode 2: Shorthand tanpa `ROW` keyword**

```sql
-- Bisa drop keyword ROW, tapi parentheses tetap perlu
SELECT (123, 'Main', 'Anytown', 'ST', '12345')::address;
-- Output: (123,Main,Anytown,ST,12345)
```

**Metode 3: String literal (TIDAK DISARANKAN)**

```sql
-- Metode string literal ribet karena nested quotes
SELECT '(123,"Main St","Any Town","ST","12345")'::address;
```

**Mengapa string literal tidak disarankan:**

- Perlu **double quoting** untuk text fields
- **Nested quotes** susah di-escape
- Error-prone dan **tidak readable**

**Rekomendasi: Gunakan `ROW()` atau shorthand parentheses.**

---

### d. Menyimpan Composite Type di Table

**Membuat table dengan composite type column:**

```sql
CREATE TABLE addresses (
  id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  addr address  -- kolom dengan tipe 'address'
);
```

**Penjelasan:**

- `id` adalah auto-incrementing primary key (syntax modern)
- `addr` adalah kolom dengan tipe **composite type `address`**
- Satu kolom `addr` menyimpan **5 fields sekaligus**

**Insert data:**

```sql
INSERT INTO addresses (addr)
VALUES (
  ROW(123, 'Main', 'Anytown', 'ST', '12345')::address
);

-- Atau menggunakan shorthand
INSERT INTO addresses (addr)
VALUES (
  (456, 'Elm St', 'Springfield', 'IL', '62701')::address
);
```

**Query data:**

```sql
SELECT * FROM addresses;
```

**Output:**

| id  | addr                                |
| --- | ----------------------------------- |
| 1   | (123,Main,Anytown,ST,12345)         |
| 2   | (456,"Elm St",Springfield,IL,62701) |

**Perhatikan:**

- Composite type ditampilkan sebagai **satu unit** dalam parentheses
- Text dengan spasi otomatis di-quote oleh PostgreSQL

---

### e. Mengakses Individual Fields dari Composite Type

**Problem: Syntax yang tricky**

```sql
-- ❌ TIDAK BEKERJA: PostgreSQL mengira 'addr' adalah table name
SELECT addr.number FROM addresses;
-- ERROR: missing FROM-clause entry for table "addr"
```

**Mengapa error?**

- PostgreSQL melihat `addr.number` dan mengira `addr` adalah **table alias**
- Karena tidak ada table bernama `addr`, maka error

**✅ Solusi: Wrap kolom dalam parentheses**

```sql
-- ✅ BEKERJA: parentheses memberitahu PostgreSQL ini composite type
SELECT (addr).number FROM addresses;
-- Output: 123

SELECT (addr).street FROM addresses;
-- Output: Main

SELECT (addr).city FROM addresses;
-- Output: Anytown
```

**Query multiple fields:**

```sql
SELECT
  id,
  (addr).number,
  (addr).street,
  (addr).city,
  (addr).state,
  (addr).zip
FROM addresses;
```

**Output:**

| id  | number | street | city        | state | zip   |
| --- | ------ | ------ | ----------- | ----- | ----- |
| 1   | 123    | Main   | Anytown     | ST    | 12345 |
| 2   | 456    | Elm St | Springfield | IL    | 62701 |

**Diagram akses field:**

```
addresses table
├── id: 1
└── addr: (123, Main, Anytown, ST, 12345)
            ↓
    Akses dengan (addr).field
            ↓
    ├── (addr).number  → 123
    ├── (addr).street  → Main
    ├── (addr).city    → Anytown
    ├── (addr).state   → ST
    └── (addr).zip     → 12345
```

---

### f. Kapan Menggunakan Composite Types vs Alternatif

**Composite Types cocok untuk:**

✅ **Extension-generated data**

```sql
-- PostGIS returns composite types
SELECT ST_GeomFromText('POINT(0 0)');
-- Returns composite type dengan x, y, srid, dll
```

✅ **Truly inseparable data groups**

```sql
-- Complex number: real dan imaginary part selalu bersama
CREATE TYPE complex AS (
  real NUMERIC,
  imag NUMERIC
);
```

✅ **Function return types**

```sql
CREATE FUNCTION get_full_address(user_id INT)
RETURNS address AS $$
  -- return composite type
$$ LANGUAGE sql;
```

**Alternatif yang lebih umum:**

**1. Discrete Columns (paling umum)**

```sql
-- ✅ Lebih jelas, lebih mudah di-query
CREATE TABLE addresses (
  id SERIAL PRIMARY KEY,
  number INT,
  street TEXT,
  city TEXT,
  state TEXT,
  zip TEXT
);

-- Query lebih mudah, tidak perlu parentheses
SELECT number, street FROM addresses WHERE city = 'Anytown';
```

**Keuntungan:**

- **Lebih readable**
- **Lebih mudah di-query** (tidak perlu syntax khusus)
- Bisa pakai **indexes** per-column
- Bisa pakai **constraints** per-column

**2. JSON/JSONB Column**

```sql
CREATE TABLE addresses (
  id SERIAL PRIMARY KEY,
  addr JSONB
);

INSERT INTO addresses (addr) VALUES (
  '{"number": 123, "street": "Main", "city": "Anytown"}'
);

-- Query dengan JSON operators
SELECT addr->>'street' FROM addresses;
```

**Keuntungan:**

- **Fleksibel**: bisa tambah/kurangi fields tanpa ALTER TABLE
- **Indexable** dengan GIN indexes
- **Semi-structured data** cocok untuk schema yang berubah

**3. Separate Table (normalization)**

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name TEXT
);

CREATE TABLE addresses (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id),
  number INT,
  street TEXT,
  city TEXT
);
```

**Keuntungan:**

- **Properly normalized**
- Multiple addresses per user
- Bisa di-query independently

**Comparison table:**

| Approach             | Readability | Flexibility | Query Ease | Use Case                         |
| -------------------- | ----------- | ----------- | ---------- | -------------------------------- |
| **Composite Type**   | Medium      | Low         | Medium     | Extension data, function returns |
| **Discrete Columns** | High        | Low         | High       | Fixed schema, frequent queries   |
| **JSON/JSONB**       | Medium      | Very High   | Medium     | Semi-structured, changing schema |
| **Separate Table**   | High        | High        | High       | Normalized data, one-to-many     |

---

### g. Real-World Use Case: PostGIS Extension

**PostGIS menggunakan composite types secara ekstensif:**

```sql
-- PostGIS geocoder returns address composite type
SELECT
  (g.addy).address,
  (g.addy).streetname,
  (g.addy).streettypeabbrev,
  (g.addy).city,
  (g.addy).state,
  (g.addy).zip
FROM
  geocode('1600 Pennsylvania Ave, Washington DC') AS g;
```

**Composite type `addy` dari PostGIS:**

- Memiliki **18+ fields** untuk komponen address
- Returned oleh geocoding functions
- User perlu tahu cara access fields dengan `(g.addy).field` syntax

**Ini adalah use case yang bagus untuk composite types karena:**

- Data structure **sudah defined** oleh extension
- User **tidak perlu modify** structure
- **Function return value** yang complex
- Semua fields **semantically related**

---

## 3. Hubungan Antar Konsep

```
┌─────────────────────────────────────────────┐
│      COMPOSITE TYPE DEFINITION              │
│   CREATE TYPE name AS (fields...)           │
└─────────────────┬───────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼────────┐    ┌─────▼──────────┐
│  INSTANTIATE   │    │  USE IN TABLE  │
│  ROW(...):type │    │  col type_name │
└───────┬────────┘    └─────┬──────────┘
        │                   │
        └─────────┬─────────┘
                  │
        ┌─────────▼──────────┐
        │   STORE IN TABLE   │
        │  INSERT VALUES(ROW)│
        └─────────┬──────────┘
                  │
        ┌─────────▼──────────┐
        │   QUERY DATA       │
        ├────────────────────┤
        │ • SELECT (col).fld │
        │ • WHERE conditions │
        └────────────────────┘
```

**Alur kerja composite types:**

1. **Define composite type** dengan `CREATE TYPE ... AS (...)`

   - Tentukan semua fields dan types
   - Seperti membuat "blueprint" data structure

2. **Use in table definition** sebagai column type

   - Kolom akan menyimpan entire composite value
   - Treated sebagai single unit

3. **Insert data** menggunakan `ROW()` atau parentheses

   - Cast ke composite type dengan `::`
   - Semua fields harus provided (atau NULL)

4. **Query data** dengan syntax khusus
   - Wrap column dalam `(...)` untuk access fields
   - Syntax: `(column_name).field_name`

**Trade-offs decision tree:**

```
Perlu store group of related fields?
    │
    ├─ Yes → Fields ALWAYS appear together?
    │        │
    │        ├─ Yes → Data from extension?
    │        │        │
    │        │        ├─ Yes → Use COMPOSITE TYPE ✓
    │        │        │
    │        │        └─ No → Schema fixed forever?
    │        │                │
    │        │                ├─ Yes → Use DISCRETE COLUMNS ✓
    │        │                │
    │        │                └─ No → Use JSONB ✓
    │        │
    │        └─ No → Use SEPARATE TABLE ✓
    │
    └─ No → Use DISCRETE COLUMNS ✓
```

---

## 4. Kesimpulan

Composite types adalah fitur PostgreSQL untuk menggabungkan multiple types menjadi **single discrete unit**, namun **jarang digunakan** dalam praktik sehari-hari.

**Key takeaways:**

**Syntax essentials:**

- **Define**: `CREATE TYPE name AS (fields...)`
- **Instantiate**: `ROW(values)::typename` atau `(values)::typename`
- **Access fields**: `(column_name).field_name` (parentheses wajib!)

**Kapan menggunakan:**

- ✅ Data dari **extensions** (seperti PostGIS)
- ✅ **Function return types** yang complex
- ✅ Truly **inseparable data groups**

**Kapan TIDAK menggunakan (hampir selalu):**

- ❌ General application data → gunakan **discrete columns**
- ❌ Semi-structured data → gunakan **JSONB**
- ❌ One-to-many relationships → gunakan **separate table**

**Limitations:**

- Tidak bisa define **constraints** di type level
- Syntax **kurang intuitif** untuk access fields
- Sulit **modify structure** setelah dibuat
- Tidak bisa **partial update** individual fields easily

**Best practice:**

> "Composite types are cool, but you probably don't need them. Default to discrete columns or JSONB unless you have a very specific reason."

Composite types adalah tool yang powerful untuk **niche use cases**, terutama saat bekerja dengan extensions atau complex function returns. Namun untuk mayoritas aplikasi, **discrete columns atau JSONB** adalah pilihan yang lebih praktis dan maintainable.
