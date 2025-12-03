# 📝 Foreign Key Constraint di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang Foreign Key Constraint di PostgreSQL, termasuk perbedaan penting antara "foreign key" (konsep) dan "foreign key constraint" (enforcement). Materi mencakup cara membuat foreign key constraint, naming conventions, composite foreign keys, dan berbagai opsi untuk menangani operasi UPDATE dan DELETE pada parent table (ON DELETE/UPDATE actions).

## 2. Konsep Utama

### a. Foreign Key vs Foreign Key Constraint - Perbedaan Fundamental

**Perbedaan krusial yang sering disalahpahami:**

| Aspek       | Foreign Key                        | Foreign Key Constraint             |
| ----------- | ---------------------------------- | ---------------------------------- |
| Definisi    | Konsep/filosofis                   | Implementasi/enforcement           |
| Sifat       | Hanya pointer/reference            | Aturan yang di-enforce database    |
| Enforcement | Tidak ada                          | Ada validasi otomatis              |
| Keharusan   | Selalu ada dalam relational design | Optional (bisa dipilih atau tidak) |

**Foreign Key (Konsep):**

- Sebuah kolom atau set kolom di child table yang "menunjuk" ke primary key di parent table
- Ini hanya konsep relasional - kolom yang berisi ID dari tabel lain
- Tidak butuh index, tidak butuh constraint
- **Selalu ada** dalam aplikasi relational database

```sql
-- Ini adalah foreign key (konsep)
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT  -- Ini foreign key, hanya kolom biasa yang berisi ID
);
```

**Foreign Key Constraint (Enforcement):**

- Aturan yang **memaksa** referential integrity
- Database akan **memvalidasi** bahwa nilai foreign key harus ada di parent table
- Mencegah orphaned rows (baris anak tanpa parent)

```sql
-- Ini adalah foreign key constraint (enforcement)
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id)  -- Ada REFERENCES = ada constraint
);
```

**Kesalahpahaman umum:**

> "I don't use foreign keys" atau "That technology doesn't support foreign keys"

**Yang sebenarnya mereka maksud:** "I don't use foreign key **constraints**"

- Foreign key (konsep) **pasti ada** kalau ada relasi antar tabel
- Yang bisa dipilih adalah apakah mau enforce dengan **constraint** atau tidak

### b. Membuat Foreign Key Constraint - Sintaks Dasar

**Setup tabel parent dan child:**

```sql
-- Parent table (states)
CREATE TABLE states (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT
);

-- Child table (cities)
CREATE TABLE cities (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id)  -- Foreign key constraint!
);
```

**Syarat mutlak:**

- **Data types HARUS match** antara foreign key dan referenced column
- Kolom yang di-reference **HARUS UNIQUE** (biasanya PRIMARY KEY atau UNIQUE constraint)

**Contoh penggunaan:**

```sql
-- Insert parent
INSERT INTO states (name) VALUES ('Texas');
-- Hasil: id = 1

-- ✅ Insert child dengan foreign key valid
INSERT INTO cities (state_id, name) VALUES (1, 'Dallas');
-- Berhasil! state_id = 1 ada di tabel states

-- ❌ Insert child dengan foreign key invalid
INSERT INTO cities (state_id, name) VALUES (2, 'Dallas');
-- Error: insert or update on table "cities" violates foreign key constraint
-- Key (state_id)=(2) is not present in table "states"
```

**Cara kerja enforcement:**

```
User: INSERT INTO cities VALUES (2, 'Dallas')
        ↓
Database: "Tunggu, kamu bilang state_id REFERENCES states(id)"
        ↓
Database: *cek tabel states untuk id = 2*
        ↓
Database: "ID 2 tidak ada! → THROW ERROR"
```

### c. Naming Conventions untuk Foreign Key

**Pattern yang direkomendasikan instruktur:**

```sql
-- Parent table: Kolom ID cukup bernama "id"
CREATE TABLE states (
    id BIGINT PRIMARY KEY,  -- Cukup "id"
    name TEXT
);

-- Child table: Foreign key = singular_table_name + "_id"
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id)  -- "state" (singular) + "_id"
);
```

**Pattern alternatif (juga valid):**

```sql
-- Parent table: Kolom ID pakai nama tabel
CREATE TABLE states (
    state_id BIGINT PRIMARY KEY,  -- "state_id" bukan "id"
    name TEXT
);

-- Child table: Foreign key juga pakai nama yang sama
CREATE TABLE cities (
    city_id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(state_id)  -- Nama sama
);
```

**Perbandingan:**

| Pattern   | Primary Key | Foreign Key | Kelebihan                    |
| --------- | ----------- | ----------- | ---------------------------- |
| Pattern 1 | `id`        | `state_id`  | Lebih ringkas, jelas mana FK |
| Pattern 2 | `state_id`  | `state_id`  | Konsisten di semua tabel     |

**Catatan penting:**

- Tidak ada pattern yang "benar mutlak"
- Pilih satu dan **konsisten** di seluruh aplikasi
- Instruktur prefer Pattern 1 (id + table_name_id)

### d. Column-Level vs Table-Level Constraint

**Column-level constraint (inline):**

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id)  -- Langsung di definisi kolom
);
```

**Table-level constraint:**

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT,
    FOREIGN KEY (state_id) REFERENCES states(id)  -- Di akhir sebagai table constraint
);
```

**Kapan menggunakan mana?**

| Jenis        | Kapan Digunakan                         |
| ------------ | --------------------------------------- |
| Column-level | Single column foreign key (paling umum) |
| Table-level  | Composite foreign key (multi-column)    |

**Keduanya functionally equivalent untuk single column!**

### e. Composite Foreign Key Constraint

**Scenario:** Parent table memiliki composite unique key

```sql
-- Parent table dengan composite unique
CREATE TABLE states (
    id BIGINT PRIMARY KEY,
    abbreviation CHAR(2),
    code INTEGER,
    name TEXT,
    UNIQUE (abbreviation, code)  -- Composite unique key
);

-- Child table dengan composite foreign key
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_abbreviation CHAR(2),
    state_code INTEGER,
    FOREIGN KEY (state_abbreviation, state_code)
        REFERENCES states(abbreviation, code)
);
```

**Aturan penting:**

- Kolom yang di-reference **HARUS memiliki UNIQUE constraint**
- Bisa PRIMARY KEY atau UNIQUE constraint
- Untuk composite: kombinasi kolom harus unique
- **Tidak bisa membuat column-level constraint untuk composite FK**

**Mengapa harus unique?**

```
Bayangkan jika tidak unique:

states table:
id | abbreviation
---|-------------
1  | TX
2  | TX  ← Duplikat!

cities table ingin reference TX:
→ Yang mana? ID 1 atau ID 2?
→ Ambiguitas! Tidak bisa menentukan parent yang tepat
→ Karena itu HARUS unique
```

### f. ON DELETE dan ON UPDATE Actions

**Opsi yang tersedia:**

| Action        | Deskripsi         | Perilaku                                                      |
| ------------- | ----------------- | ------------------------------------------------------------- |
| `NO ACTION`   | Default           | Cegah delete/update parent jika ada child                     |
| `RESTRICT`    | Hampir sama       | Cegah delete/update, tidak bisa deferred                      |
| `CASCADE`     | Cascade operation | Delete/update parent → delete/update child juga               |
| `SET NULL`    | Set to NULL       | Delete/update parent → set foreign key child jadi NULL        |
| `SET DEFAULT` | Set to default    | Delete/update parent → set foreign key child ke default value |

#### ON DELETE NO ACTION (Default)

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id)
    -- Default: ON DELETE NO ACTION
);

INSERT INTO states (name) VALUES ('Texas');  -- id = 1
INSERT INTO cities (state_id, name) VALUES (1, 'Dallas');

-- Coba delete parent
DELETE FROM states WHERE id = 1;
-- ❌ Error: update or delete on table "states" violates foreign key constraint
-- Detail: Key (id)=(1) is still referenced from table "cities"
```

**Perilaku:** Tidak bisa delete parent jika masih ada child yang reference

#### ON DELETE RESTRICT

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id) ON DELETE RESTRICT
);

DELETE FROM states WHERE id = 1;
-- ❌ Error: sama seperti NO ACTION
```

**Perbedaan NO ACTION vs RESTRICT:**

- `NO ACTION`: Check bisa di-defer ke akhir transaction
- `RESTRICT`: Check langsung dilakukan, tidak bisa di-defer
- **Pada praktiknya:** Hasilnya sama - parent tidak bisa di-delete

#### ON DELETE CASCADE (⚠️ BERBAHAYA!)

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id) ON DELETE CASCADE
);

INSERT INTO states (name) VALUES ('Texas');  -- id = 1
INSERT INTO cities (state_id, name) VALUES (1, 'Dallas');

SELECT * FROM cities;
-- id | name   | state_id
-- ---|--------|----------
-- 1  | Dallas | 1

-- Delete parent
DELETE FROM states WHERE id = 1;
-- ✅ Berhasil!

SELECT * FROM cities;
-- (empty result)
-- Dallas juga ikut terhapus!
```

**⚠️ Peringatan keras dari instruktur:**

> "I feel uncomfortable about this because you can imagine a cascading cascade of cascades"

**Skenario berbahaya:**

```
DELETE account
  ↓ CASCADE
  DELETE projects (100 rows)
    ↓ CASCADE
    DELETE tasks (10,000 rows)
      ↓ CASCADE
      DELETE comments (100,000 rows)
        ↓ CASCADE
        DELETE attachments (1,000,000 rows)
          ↓ CASCADE
          ...

Result: 1 DELETE query → Millions of rows deleted!
```

**Rekomendasi instruktur:**

- **Prefer:** `ON DELETE RESTRICT` atau default (`NO ACTION`)
- **Avoid:** `ON DELETE CASCADE` kecuali sangat yakin dengan konsekuensinya
- Jika perlu delete cascade, lebih baik handle di application level dengan explicit deletes

#### ON DELETE SET NULL

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id) ON DELETE SET NULL
    -- Kolom state_id HARUS nullable!
);

INSERT INTO states (name) VALUES ('Texas');  -- id = 1
INSERT INTO cities (state_id, name) VALUES (1, 'Dallas');

DELETE FROM states WHERE id = 1;
-- ✅ Berhasil!

SELECT * FROM cities;
-- id | name   | state_id
-- ---|--------|----------
-- 1  | Dallas | NULL      ← Di-set NULL
```

**Use case:**

- Ketika foreign key column nullable
- Ingin keep orphaned rows untuk cleanup nanti
- Menghindari cascade delete

**Syarat:** Kolom foreign key harus nullable (tidak bisa `NOT NULL`)

#### ON DELETE SET DEFAULT

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT DEFAULT 1 REFERENCES states(id) ON DELETE SET DEFAULT
);

DELETE FROM states WHERE id = 1;
-- state_id akan di-set ke default value (1)
-- Tapi ini akan error jika id = 1 tidak ada!
```

**Use case:** Jarang digunakan, biasanya untuk default "unknown" state

### g. ON UPDATE Actions

**Sintaks sama dengan ON DELETE:**

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT
);
```

**Perilaku:**

- `ON UPDATE CASCADE`: Jika parent ID berubah, child ID ikut berubah
- `ON UPDATE RESTRICT`: Tidak bisa update parent ID jika ada child

**Catatan praktis:**

- ON UPDATE jarang digunakan karena primary key biasanya tidak berubah
- Jika pakai generated identity atau UUID, kemungkinan update sangat kecil

### h. Contoh Lengkap dengan Best Practices

```sql
-- Parent table
CREATE TABLE states (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    abbreviation CHAR(2) NOT NULL UNIQUE
);

-- Child table dengan foreign key constraint
CREATE TABLE cities (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    population INTEGER,
    state_id BIGINT NOT NULL REFERENCES states(id)
        ON DELETE RESTRICT      -- Tidak bisa delete state jika ada cities
        ON UPDATE CASCADE,      -- Update state.id → cities.state_id ikut update
    CONSTRAINT cities_state_fk
        FOREIGN KEY (state_id) REFERENCES states(id)  -- Named constraint
);
```

**Best practices yang diterapkan:**

1. ✅ NOT NULL pada foreign key (jika memang harus ada)
2. ✅ ON DELETE RESTRICT (aman, tidak ada cascade tak terduga)
3. ✅ ON UPDATE CASCADE (jarang perlu, tapi aman)
4. ✅ Named constraint untuk debugging lebih mudah

## 3. Hubungan Antar Konsep

### Alur Pemahaman:

```
1. Konsep Relational Database
   ↓
   Tables saling berhubungan via ID

2. Foreign Key (Konsep)
   ↓
   Kolom yang berisi ID dari tabel lain
   ↓
   Ini SELALU ada dalam relational design

3. Foreign Key Constraint (Enforcement)
   ↓
   OPTIONAL: Apakah mau enforce referential integrity?
   ↓
   Jika ya → tambahkan REFERENCES clause

4. Enforcement Behavior
   ├─ ON DELETE: Apa yang terjadi saat parent dihapus?
   │  ├─ NO ACTION/RESTRICT: Cegah delete parent
   │  ├─ CASCADE: Delete child juga (BAHAYA!)
   │  ├─ SET NULL: Set foreign key child jadi NULL
   │  └─ SET DEFAULT: Set foreign key child ke default
   │
   └─ ON UPDATE: Apa yang terjadi saat parent ID berubah?
      └─ (Jarang digunakan, biasanya CASCADE atau RESTRICT)
```

### Diagram Referential Integrity:

```
WITHOUT Foreign Key Constraint:
================================
states                cities
id | name              id | name   | state_id
---|------             ---|--------|----------
1  | Texas             1  | Dallas | 1
                       2  | Austin | 99  ← Orphaned! (state_id 99 tidak ada)

Database: "Ok, I'll insert it" ✅ (No validation)


WITH Foreign Key Constraint:
============================
states                cities (state_id REFERENCES states(id))
id | name              id | name   | state_id
---|------             ---|--------|----------
1  | Texas             1  | Dallas | 1

INSERT cities VALUES (2, 'Austin', 99)
                       ↓
Database: "Wait, let me check states table..."
                       ↓
Database: "ID 99 not found → REJECT" ❌
```

### Decision Tree: Memilih ON DELETE Action

```
Apakah child rows masih relevan tanpa parent?
├─ YA (contoh: archived data, historical records)
│  └─ Gunakan: SET NULL atau SET DEFAULT
│     └─ Cleanup orphaned rows secara manual/batch job
│
└─ TIDAK (child tidak ada artinya tanpa parent)
   ├─ Apakah delete parent sering terjadi?
   │  ├─ YA → CASCADE mungkin OK (tapi hati-hati!)
   │  │  └─ ⚠️ Pastikan tidak ada cascade chain yang panjang
   │  │
   │  └─ TIDAK → RESTRICT/NO ACTION (safest!)
   │     └─ Force application to explicitly delete children first
   │
   └─ Default choice: RESTRICT atau NO ACTION
```

### Hierarchy of Safety:

```
PALING AMAN (Recommended)
↓
1. ON DELETE RESTRICT / NO ACTION
   - Tidak ada surprise deletes
   - Explicit handling di application

2. ON DELETE SET NULL
   - Orphaned rows tapi masih ada
   - Bisa cleanup nanti

3. ON DELETE CASCADE
   - Convenient tapi DANGEROUS
   - Bisa delete millions of rows accidentally
↓
PALING BERBAHAYA
```

### Hubungan dengan Konsep Lain:

**Foreign Key + Indexing (akan dibahas di modul joins):**

```sql
-- Foreign key constraint ≠ Index
CREATE TABLE cities (
    state_id BIGINT REFERENCES states(id)  -- Ada constraint
);

-- Tapi belum ada index!
-- Untuk performa joins, perlu add index manually:
CREATE INDEX idx_cities_state_id ON cities(state_id);
```

**Foreign Key + UNIQUE Constraint:**

```sql
-- Yang di-reference HARUS UNIQUE
CREATE TABLE states (
    id BIGINT PRIMARY KEY,        -- PRIMARY KEY = UNIQUE
    abbreviation CHAR(2) UNIQUE   -- Explicit UNIQUE
);

-- Bisa reference ke kolom manapun yang UNIQUE
CREATE TABLE cities (
    state_id BIGINT REFERENCES states(id),           -- ✅ OK
    state_abbr CHAR(2) REFERENCES states(abbreviation) -- ✅ OK juga
);
```

## 4. Kesimpulan

**Key Takeaways:**

1. **Foreign Key ≠ Foreign Key Constraint:**

   - Foreign Key = Konsep relasional (selalu ada)
   - Foreign Key Constraint = Enforcement (optional, tapi recommended)

2. **Membuat Foreign Key Constraint:**

   ```sql
   column_name TYPE REFERENCES parent_table(column)
   ```

3. **Syarat mutlak:**

   - Data types harus match
   - Kolom yang di-reference harus UNIQUE (PRIMARY KEY atau UNIQUE constraint)

4. **Naming conventions:**

   - Pattern 1: `id` di parent, `table_name_id` di child (recommended)
   - Pattern 2: `table_name_id` di semua tabel
   - Yang penting: **konsisten!**

5. **Column-level vs Table-level:**

   - Column-level: Untuk single column FK
   - Table-level: Untuk composite FK (wajib)

6. **ON DELETE Actions:**

   - `NO ACTION` / `RESTRICT`: Default, safest (recommended)
   - `CASCADE`: Convenient tapi dangerous (hati-hati!)
   - `SET NULL`: Orphaned rows untuk cleanup nanti
   - `SET DEFAULT`: Jarang digunakan

7. **Best Practice dari instruktur:**
   - ✅ Gunakan foreign key constraints untuk referential integrity
   - ✅ Prefer ON DELETE RESTRICT atau NO ACTION
   - ⚠️ Hindari ON DELETE CASCADE (atau gunakan dengan sangat hati-hati)
   - ✅ Jika perlu cascade, handle di application level dengan explicit deletes

**Manfaat Foreign Key Constraint:**

- Menjamin referential integrity
- Mencegah orphaned rows
- Database-level enforcement (tidak perlu handle di application)
- Self-documenting relationships

**Yang akan dibahas nanti:**

- Indexing foreign keys untuk performa joins (modul joins)
- Composite foreign keys detail
- Advanced scenarios dengan multi-table relationships

**Prinsip emas:** "Foreign key constraints give me referential integrity, but be careful with CASCADE - it can explode into millions of deletes!"
