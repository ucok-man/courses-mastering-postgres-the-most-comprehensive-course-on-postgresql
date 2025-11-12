# 📝 JSON dan JSONB di PostgreSQL

## 1. Ringkasan Singkat

Materi ini membahas tentang menyimpan data **JSON** di PostgreSQL, topik yang cukup kontroversial di dunia database relational. PostgreSQL menyediakan dua tipe data untuk JSON: `JSON` dan `JSONB`. Video ini menjelaskan perbedaan keduanya, kapan sebaiknya menggunakan JSON di database, pedoman untuk memutuskan apakah data harus disimpan sebagai JSON atau dipecah menjadi kolom-kolom terpisah, serta preview singkat tentang operasi-operasi yang bisa dilakukan pada data JSON.

## 2. Konsep Utama

### a. Kontroversi: Haruskah JSON Disimpan di Database Relational?

**Dua kubu ekstrem:**

```
Kubu 1 (Puritan):                    Kubu 2 (Liberal):
"JANGAN PERNAH simpan JSON           "Treat Postgres seperti MongoDB
di relational database!"             dengan primary key + JSON blob!"
         |                                      |
         └──────────────┬──────────────────────┘
                        |
                   Pendekatan Tengah ✅
           "JSON berguna untuk use case tertentu"
```

**Posisi yang direkomendasikan:**

- JSON **sangat berguna** dan tidak ada yang salah dengan menyimpannya di PostgreSQL
- PostgreSQL memiliki **banyak fitur** untuk manipulasi JSON (querying, indexing, updating)
- Yang penting adalah **tahu kapan** menggunakan JSON dan kapan menggunakan kolom terpisah

### b. Pedoman Kapan Menggunakan JSON vs Kolom Terpisah

#### **1. Pertimbangan Access Pattern**

**❌ Jangan gunakan JSON jika:**

- Kamu **terus-menerus query** field tertentu di dalam JSON
- Field tersebut adalah **primary lookup key** (misalnya email user)

**✅ Gunakan JSON jika:**

- Seluruh blob di-update sekaligus
- Jarang perlu query field internal
- Data bersifat semi-structured atau flexible

**Contoh kasus:**

```sql
-- ❌ KURANG OPTIMAL (email sering di-query)
CREATE TABLE users_bad (
    id BIGINT PRIMARY KEY,
    data JSONB  -- berisi: {email, name, address, preferences, ...}
);

-- ✅ LEBIH BAIK
CREATE TABLE users_good (
    id BIGINT PRIMARY KEY,
    email TEXT NOT NULL,              -- Top-level column
    name TEXT,                         -- Top-level column
    preferences JSONB                  -- JSON untuk data yang jarang di-query
);
```

**Solusi tengah: Generated Columns**

- Kamu bisa tetap simpan JSON blob lengkap
- Tapi ekstrak field penting ke top-level column
- (Akan dibahas detail di materi generated columns)

#### **2. Pertimbangan Schema Structure**

**Tanda harus pisah ke kolom terpisah:**

- Schema **rigid** dan **well-defined**
- Structure sudah dipahami dengan baik
- Field-field di-update **secara independen**

**Tanda bisa tetap JSON:**

- Schema **flexible** atau **evolving**
- Seluruh document di-update **sekaligus**
- PostgreSQL punya **JSON patching support** (bisa update sebagian)

**Contoh:**

```sql
-- Schema rigid → Pecah ke kolom
CREATE TABLE products_structured (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL,
    category TEXT
);

-- Schema flexible → Simpan sebagai JSON
CREATE TABLE products_flexible (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    metadata JSONB  -- attributes yang bervariasi per product type
);
```

#### **3. Pertimbangan Ukuran Document**

**Batasan teknis:**

- Column `JSONB` bisa menyimpan hingga **255 MB** per document
- Tapi "bisa" ≠ "sebaiknya"

**Rekomendasi:**

- Jika JSON document sangat besar (puluhan MB), **performance akan menurun**
- Lebih baik **pecah menjadi beberapa JSON column** yang lebih kecil
- Akses menjadi lebih **scoped** dan **discrete**

**Contoh:**

```sql
-- ❌ Satu JSON blob besar (30 MB)
CREATE TABLE user_data_bad (
    user_id BIGINT PRIMARY KEY,
    all_data JSONB  -- profile + preferences + history + analytics
);

-- ✅ Dipecah ke beberapa JSON column
CREATE TABLE user_data_good (
    user_id BIGINT PRIMARY KEY,
    profile JSONB,        -- ~1 MB
    preferences JSONB,    -- ~500 KB
    activity_log JSONB,   -- ~10 MB
    analytics JSONB       -- ~5 MB
);
```

### c. JSON vs JSONB: Perbedaan Fundamental

PostgreSQL menyediakan **dua tipe data** untuk JSON dengan karakteristik berbeda:

#### **Perbandingan JSON vs JSONB**

| Aspek              | JSON                      | JSONB                                |
| ------------------ | ------------------------- | ------------------------------------ |
| **Storage**        | Text format (lebih kecil) | Binary format (lebih besar)          |
| **Parsing**        | Setiap kali diakses       | Sekali saat insert                   |
| **Performance**    | Lambat untuk operasi      | Cepat untuk operasi                  |
| **Whitespace**     | Dipertahankan             | Dihilangkan                          |
| **Key order**      | Dipertahankan             | Tidak dijamin                        |
| **Duplicate keys** | Dipertahankan             | Otomatis di-overwrite                |
| **Indexing**       | Tidak bisa                | Bisa di-index                        |
| **Use case**       | Legacy compatibility      | **Recommended untuk semua use case** |

#### **Contoh Perbedaan Storage Size**

```sql
-- Contoh sederhana
SELECT
    pg_column_size('{"a": 1}'::JSON) as json_size,    -- 5 bytes
    pg_column_size('{"a": 1}'::JSONB) as jsonb_size;  -- 20 bytes

-- Contoh lebih besar (overhead ter-amortisasi)
SELECT
    pg_column_size('{"a": "hello world"}'::JSON) as json_size,
    pg_column_size('{"a": "hello world"}'::JSONB) as jsonb_size;
-- Perbedaan mengecil untuk document lebih besar
```

**Mengapa JSONB lebih besar?**

- Sudah di-**parsed** dan di-**deconstruct**
- Disimpan dalam **binary representation**
- Menyimpan **metadata tambahan** untuk mempercepat query
- Overhead ini worth it untuk performance gain

#### **Contoh Perbedaan Behavior: Whitespace**

```sql
-- JSON: mempertahankan whitespace
SELECT '{"a":    "hello world"}'::JSON;
-- Output: {"a":    "hello world"}  (whitespace dipertahankan)

-- JSONB: menghilangkan whitespace
SELECT '{"a":    "hello world"}'::JSONB;
-- Output: {"a":"hello world"}  (whitespace dihilangkan, normalized)
```

#### **Contoh Perbedaan Behavior: Duplicate Keys**

```sql
-- JSON: mempertahankan duplicate keys
SELECT '{"a": 1, "a": 2}'::JSON;
-- Output: {"a": 1, "a": 2}  (kedua key ada)

-- JSONB: mengikuti JSON spec (last key wins)
SELECT '{"a": 1, "a": 2}'::JSONB;
-- Output: {"a":2}  (key pertama di-overwrite)
```

#### **Contoh Perbedaan Behavior: Key Ordering**

```sql
-- JSON: mempertahankan urutan key
SELECT '{"z": 1, "a": 2, "m": 3}'::JSON;
-- Output: {"z": 1, "a": 2, "m": 3}  (urutan sama)

-- JSONB: tidak menjamin urutan (bisa berubah)
SELECT '{"z": 1, "a": 2, "m": 3}'::JSONB;
-- Output: urutan bisa berbeda, tidak terjamin
```

### d. Kapan Menggunakan JSON (bukan JSONB)?

**⚠️ Use case JSON (sangat jarang):**

1. **Legacy systems** yang bergantung pada:

   - Whitespace preservation
   - Duplicate keys (walau tidak recommended)
   - Key ordering (walau melanggar JSON spec)

2. **Validasi saja** tanpa perlu operasi
   - Hanya menyimpan dan mengembalikan text as-is
   - Perlu validasi bahwa input adalah valid JSON

**✅ Recommended: Gunakan JSONB untuk semua use case modern**

- Lebih cepat untuk operasi
- Bisa di-index
- Mengikuti JSON spec dengan benar
- Support semua fitur PostgreSQL untuk JSON

### e. Preview: Operasi pada JSONB

PostgreSQL menyediakan banyak operator dan fungsi untuk bekerja dengan JSONB:

#### **Ekstraksi Value dengan Operator `->`**

```sql
-- Ekstrak value (hasil masih JSONB)
SELECT '{"string": "hello world", "number": 42}'::JSONB -> 'string';
-- Output: "hello world"  (masih ada quotes, tipe JSONB)
```

#### **Ekstraksi Text dengan Operator `->>`**

```sql
-- Ekstrak sebagai text (unquoted)
SELECT '{"string": "hello world", "number": 42}'::JSONB ->> 'string';
-- Output: hello world  (tanpa quotes, tipe TEXT)
```

#### **Akses Nested Objects**

```sql
SELECT '{
  "objects": {
    "key": "value"
  },
  "array": [1, 2, 3]
}'::JSONB -> 'objects' -> 'key';
-- Output: "value"
```

#### **Akses Array Elements**

```sql
-- Menggunakan index array
SELECT '{
  "array": [10, 20, 30]
}'::JSONB -> 'array' -> 2;
-- Output: 30  (index dimulai dari 0)
```

#### **Path Expressions dengan `#>` Operator**

```sql
-- Akses nested structure dengan path
SELECT '{
  "objects": {
    "nested": {
      "key": "deep value"
    }
  }
}'::JSONB #> '{objects,nested,key}';
-- Output: "deep value"
```

**Operasi lain yang akan dibahas di module JSON:**

- Check keberadaan keys
- Check overlap dalam structure
- Array operations
- JSON patching/updating
- Indexing strategies

### f. Best Practices untuk JSON di PostgreSQL

**Checklist keputusan:**

```
┌─ Apakah field ini sering di-query? ───► Ya ──► Top-level column
│                                         │
│                                         No
│                                         │
├─ Apakah schema rigid dan well-defined? ─► Ya ──► Pertimbangkan kolom terpisah
│                                         │
│                                         No
│                                         │
├─ Apakah field di-update independen? ───► Ya ──► Pertimbangkan kolom terpisah
│                                         │
│                                         No
│                                         │
├─ Apakah document sangat besar (>10MB)?──► Ya ──► Pecah ke multiple JSON columns
│                                         │
│                                         No
│                                         │
└─ Gunakan JSONB untuk flexibility ──────────────► ✅
```

**Golden Rules:**

1. **Gunakan JSONB** (bukan JSON) untuk hampir semua kasus
2. **Top-level columns** untuk data yang sering di-query
3. **JSON columns** untuk flexible/semi-structured data
4. **Pertimbangkan ukuran** document (jangan terlalu besar)
5. **PostgreSQL bekerja terbaik** dengan typed columns yang discrete

## 3. Hubungan Antar Konsep

Filosofi penyimpanan data di PostgreSQL:

```
Prinsip Fundamental:
"Pick the data type that most accurately represents reality"
                    |
        ┌───────────┴───────────┐
        |                       |
   Structured Data        Semi-structured Data
   (Known schema)         (Flexible schema)
        |                       |
   ┌────┴────┐            ┌────┴────┐
   |         |            |         |
Discrete   JSON for   JSONB for  Legacy
Columns   metadata   flexibility  (JSON)
```

**Alur keputusan:**

1. **Mulai dengan struktur relational** → Kolom-kolom typed yang discrete
2. **Identifikasi data flexible** → Kandidat untuk JSON
3. **Evaluasi access patterns** → Sering di-query? → Extract ke top-level
4. **Pilih JSONB** (bukan JSON) untuk modern use cases
5. **Index jika perlu** → PostgreSQL support indexing JSONB

**Trade-offs:**

```
JSON Storage:
   Advantages                      Disadvantages
   ✅ Flexibility                  ❌ Kurang performant untuk frequent queries
   ✅ Schema evolution             ❌ Kehilangan type safety
   ✅ Nested structures            ❌ Kompleksitas query meningkat
   ✅ Compact untuk some data      ❌ Indexing lebih kompleks

Top-level Columns:
   Advantages                      Disadvantages
   ✅ Performance optimal          ❌ Less flexible
   ✅ Type safety                  ❌ Schema changes perlu migration
   ✅ Simple queries               ❌ Sulit untuk nested data
   ✅ Easy indexing                ❌ Banyak NULL jika data sparse
```

**Skenario ideal untuk JSONB:**

- User preferences/settings
- Product attributes (varies by category)
- Metadata/tags
- API responses yang disimpan
- Audit logs
- Configuration data

**Skenario ideal untuk kolom terpisah:**

- Primary identifiers (email, username)
- Frequently queried fields
- Fields yang di-update independen
- Foreign keys
- Core business data

## 4. Kesimpulan

- PostgreSQL menyediakan **dua tipe data JSON**: `JSON` dan `JSONB`
- **JSONB adalah pilihan yang direkomendasikan** untuk hampir semua use case:
  - Stored dalam **binary format** (sudah parsed)
  - **Lebih cepat** untuk operasi query/update
  - **Bisa di-index** untuk performance
  - Mengikuti **JSON spec** dengan benar (no duplicate keys)
- **JSON type** hanya untuk legacy compatibility (preserves whitespace, key order, duplicates)
- **Pedoman keputusan** menyimpan sebagai JSON vs kolom terpisah:
  1. **Access patterns** → Sering di-query? → Top-level column
  2. **Schema rigidity** → Rigid & well-defined? → Kolom terpisah
  3. **Update patterns** → Updated independently? → Kolom terpisah
  4. **Document size** → Sangat besar? → Pecah ke multiple columns
- PostgreSQL memiliki **fitur lengkap** untuk manipulasi JSON:
  - Extraction operators (`->`, `->>`, `#>`)
  - Path expressions
  - Indexing support
  - JSON patching
  - Array operations
- **Best practice**: Kombinasikan typed columns untuk data core dengan JSONB untuk flexibility

**Template yang direkomendasikan:**

```sql
CREATE TABLE modern_table (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

    -- Core fields: typed columns
    email TEXT NOT NULL,
    username TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Flexible data: JSONB
    preferences JSONB,
    metadata JSONB,

    -- Indexes untuk JSONB jika perlu
    -- (akan dibahas di module indexing)
);
```
