# 📝 UNIQUE Constraint di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang constraint UNIQUE di PostgreSQL, yang memastikan tidak ada nilai duplikat dalam satu kolom atau kombinasi beberapa kolom. Materi mencakup perbedaan antara PRIMARY KEY dan UNIQUE, perilaku khusus UNIQUE dengan nilai NULL, serta berbagai cara mendeklarasikan constraint UNIQUE termasuk composite unique constraint.

## 2. Konsep Utama

### a. PRIMARY KEY vs UNIQUE

**PRIMARY KEY = NOT NULL + UNIQUE**

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
    -- PRIMARY KEY otomatis menambahkan:
    -- 1. NOT NULL constraint
    -- 2. UNIQUE constraint
);
```

**Poin penting:**
- PRIMARY KEY adalah gabungan dari dua constraint sekaligus
- Setiap tabel biasanya hanya memiliki satu PRIMARY KEY
- Namun, kita bisa menambahkan constraint UNIQUE pada kolom-kolom lain

### b. UNIQUE Constraint Dasar

**Sintaks sederhana:**

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_number TEXT UNIQUE  -- Nilai harus unik
);

-- Insert pertama berhasil
INSERT INTO products VALUES (DEFAULT, 'ABC123');  -- ✅ Berhasil

-- Insert kedua dengan nilai sama akan gagal
INSERT INTO products VALUES (DEFAULT, 'ABC123');  -- ❌ Error: already exists
```

**Error message:**
```
ERROR: duplicate key value violates unique constraint
DETAIL: Key (product_number)=(ABC123) already exists.
```

### c. UNIQUE dan NULL - Perilaku Khusus

**Perilaku default: NULL dianggap DISTINCT (berbeda satu sama lain)**

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_number TEXT UNIQUE  -- Nullable by default
);

-- Semua insert ini BERHASIL karena NULL ≠ NULL
INSERT INTO products VALUES (DEFAULT, NULL);  -- ✅ Berhasil
INSERT INTO products VALUES (DEFAULT, NULL);  -- ✅ Berhasil juga!
INSERT INTO products VALUES (DEFAULT, NULL);  -- ✅ Masih berhasil!

SELECT * FROM products;
```

**Hasil:**
```
id | product_number
---|---------------
1  | NULL
2  | NULL
3  | NULL
```

**Mengapa ini bisa terjadi?**
Ingat prinsip dari video sebelumnya: **NULL = unknown value**

- NULL ≠ NULL (karena "unknown ≠ unknown")
- Database tidak bisa menentukan apakah dua nilai NULL itu sama atau berbeda
- Maka semua NULL dianggap distinct (berbeda)

### d. Menangani NULL dalam UNIQUE Constraint

**Ada 3 opsi untuk menangani NULL:**

#### Opsi 1: Membuat kolom NOT NULL

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_number TEXT NOT NULL UNIQUE  -- Tidak boleh NULL
);

-- Ini akan error
INSERT INTO products VALUES (DEFAULT, NULL);  -- ❌ Error: NOT NULL violation
```

#### Opsi 2: UNIQUE NULLS NOT DISTINCT

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_number TEXT UNIQUE NULLS NOT DISTINCT
);

-- Insert NULL pertama berhasil
INSERT INTO products VALUES (DEFAULT, NULL);  -- ✅ Berhasil

-- Insert NULL kedua GAGAL
INSERT INTO products VALUES (DEFAULT, NULL);  -- ❌ Error: duplicate key
```

**Karakteristik NULLS NOT DISTINCT:**
- Hanya mengizinkan **SATU nilai NULL** di kolom tersebut
- NULL diperlakukan seolah-olah sama dengan NULL lainnya (perilaku non-standard)
- Berguna untuk kasus: "boleh ada satu nilai unknown, tapi sisanya harus distinct"

**Diagram perilaku:**

```
Default UNIQUE:
NULL, NULL, NULL, 'A', 'B' → ✅ Valid (multiple NULLs allowed)

UNIQUE NULLS NOT DISTINCT:
NULL, 'A', 'B' → ✅ Valid (one NULL allowed)
NULL, NULL, 'A' → ❌ Invalid (duplicate NULL)

NOT NULL UNIQUE:
'A', 'B', 'C' → ✅ Valid (no NULLs at all)
NULL, 'A' → ❌ Invalid (NULL not allowed)
```

### e. Composite UNIQUE Constraint

**UNIQUE constraint bisa diterapkan pada kombinasi beberapa kolom:**

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    brand TEXT NOT NULL,
    product_number TEXT NOT NULL,
    UNIQUE (brand, product_number)  -- Kombinasi harus unik
);

-- Insert dengan kombinasi berbeda
INSERT INTO products VALUES (DEFAULT, 'ABC', '123');  -- ✅ Berhasil
INSERT INTO products VALUES (DEFAULT, 'DEF', '123');  -- ✅ Berhasil (brand berbeda)
INSERT INTO products VALUES (DEFAULT, 'ABC', '456');  -- ✅ Berhasil (number berbeda)

-- Insert dengan kombinasi yang sama
INSERT INTO products VALUES (DEFAULT, 'ABC', '123');  -- ❌ Error: composite unique violation
```

**Hasil query:**

```sql
SELECT * FROM products;
```

```
id | brand | product_number
---|-------|---------------
1  | ABC   | 123
2  | DEF   | 123           ← 123 muncul 2x, tapi brand berbeda ✅
3  | ABC   | 456           ← ABC muncul 2x, tapi number berbeda ✅
```

**Poin penting:**
- Kolom individual boleh memiliki duplikat
- Yang harus unik adalah **kombinasi** dari semua kolom dalam constraint

### f. Deklarasi UNIQUE sebagai Table Constraint

**Ada beberapa cara mendeklarasikan UNIQUE:**

#### Cara 1: Inline Column Constraint (sudah dibahas)

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    product_number TEXT UNIQUE  -- Inline
);
```

#### Cara 2: Table Constraint

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    product_number TEXT,
    UNIQUE (product_number)  -- Di akhir sebagai table constraint
);
```

#### Cara 3: Table Constraint dengan Nama

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    product_number TEXT,
    CONSTRAINT must_be_unique UNIQUE (product_number)  -- Dengan nama custom
);
```

**Error message dengan constraint name:**
```
ERROR: duplicate key value violates unique constraint "must_be_unique"
```

**Kapan menggunakan cara mana?**

| Cara | Kapan Digunakan |
|------|-----------------|
| Inline | Untuk single column, lebih ringkas |
| Table constraint | Untuk composite unique, atau jika ingin konsisten dengan constraint lain |
| Named constraint | Jika ingin error message lebih deskriptif |

### g. Ringkasan Opsi UNIQUE

**Tabel perbandingan semua opsi:**

```sql
-- Opsi 1: Basic unique (multiple NULLs allowed)
product_number TEXT UNIQUE

-- Opsi 2: No NULLs at all
product_number TEXT NOT NULL UNIQUE

-- Opsi 3: Only one NULL allowed
product_number TEXT UNIQUE NULLS NOT DISTINCT

-- Opsi 4: Composite unique
UNIQUE (brand, product_number)

-- Opsi 5: Table constraint
UNIQUE (product_number)

-- Opsi 6: Named constraint
CONSTRAINT constraint_name UNIQUE (product_number)

-- Opsi 7: Composite dengan nulls not distinct
UNIQUE NULLS NOT DISTINCT (brand, product_number)
```

## 3. Hubungan Antar Konsep

### Alur Pemahaman:

```
1. PRIMARY KEY
   ↓
   Terdiri dari: NOT NULL + UNIQUE
   
2. UNIQUE Constraint
   ↓
   Bisa diterapkan pada kolom lain selain primary key
   
3. UNIQUE dengan NULL
   ↓
   Default: Multiple NULLs allowed (karena NULL ≠ NULL)
   
4. Solusi untuk NULL
   ├─ NOT NULL UNIQUE → No NULLs allowed
   └─ NULLS NOT DISTINCT → One NULL allowed
   
5. Composite UNIQUE
   ↓
   Kombinasi kolom harus unik (bukan individual)
   
6. Deklarasi
   ├─ Inline column constraint
   ├─ Table constraint
   └─ Named constraint
```

### Diagram Decision Tree:

```
Apakah butuh constraint UNIQUE?
├─ Pada PRIMARY KEY?
│  └─ Tidak perlu deklarasi UNIQUE (sudah otomatis)
│
└─ Pada kolom lain?
   ├─ Single column?
   │  ├─ Boleh NULL?
   │  │  ├─ Ya, multiple NULLs OK → TEXT UNIQUE
   │  │  ├─ Ya, tapi max 1 NULL → TEXT UNIQUE NULLS NOT DISTINCT
   │  │  └─ Tidak boleh NULL → TEXT NOT NULL UNIQUE
   │  │
   │  └─ Inline atau table constraint?
   │     ├─ Inline → TEXT UNIQUE
   │     └─ Table → UNIQUE (column_name)
   │
   └─ Multiple columns?
      └─ UNIQUE (col1, col2, ...)
```

### Hubungan dengan Konsep NULL:

Pemahaman UNIQUE constraint sangat bergantung pada pemahaman NULL dari video sebelumnya:

- **NULL = unknown value**
- **NULL ≠ NULL** dalam logika database
- Ini menjelaskan mengapa default UNIQUE mengizinkan multiple NULLs
- `NULLS NOT DISTINCT` adalah override dari perilaku default ini

## 4. Kesimpulan

**Key Takeaways:**

1. **PRIMARY KEY = NOT NULL + UNIQUE** secara otomatis
2. **UNIQUE constraint bisa ditambahkan pada kolom atau kombinasi kolom mana pun**
3. **Perilaku default UNIQUE dengan NULL:**
   - Multiple NULLs diizinkan (karena NULL ≠ NULL)
   - Ini mungkin atau mungkin tidak sesuai kebutuhan bisnis
4. **Tiga strategi menangani NULL:**
   - `NOT NULL UNIQUE`: Tidak boleh NULL sama sekali
   - `UNIQUE`: Multiple NULLs allowed (default)
   - `UNIQUE NULLS NOT DISTINCT`: Maksimal satu NULL
5. **Composite UNIQUE:** Kombinasi kolom harus unik, bukan kolom individual
6. **Cara deklarasi:** Inline, table constraint, atau named constraint - semuanya valid

**Best Practice:**
- Pertimbangkan apakah kolom boleh NULL atau tidak terlebih dahulu
- Untuk sebagian besar kasus, `NOT NULL UNIQUE` adalah pilihan paling jelas
- Gunakan `NULLS NOT DISTINCT` hanya jika ada use case spesifik untuk "satu nilai unknown"
- Gunakan composite unique untuk business rules yang melibatkan kombinasi kolom
- Beri nama constraint jika ingin error message lebih deskriptif