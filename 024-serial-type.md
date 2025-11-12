# 📝 Serial Type di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang tipe data **serial** di PostgreSQL, yang sebenarnya adalah **alias** atau shortcut yang membuat kolom auto-incrementing. Serial banyak digunakan untuk primary key, namun sejak PostgreSQL 10, ada cara yang lebih baik (identity columns). Video ini menjelaskan cara kerja serial, hubungannya dengan sequences, mengapa bisa ada gaps, dan bagaimana menggunakannya dengan benar.

## 2. Konsep Utama

### a. Apa itu Serial?

**Serial bukan tipe data sebenarnya** - ini adalah alias yang di balik layar melakukan beberapa hal sekaligus.

**Deklarasi sederhana:**

```sql
CREATE TABLE example (
    id SERIAL
);
```

**Apa yang sebenarnya terjadi di balik layar:**

```sql
-- 1. Membuat sequence
CREATE SEQUENCE example_id_seq;

-- 2. Membuat kolom dengan tipe INTEGER
CREATE TABLE example (
    id INTEGER NOT NULL
    DEFAULT nextval('example_id_seq')
);

-- 3. Mengatur ownership sequence
ALTER SEQUENCE example_id_seq
OWNED BY example.id;
```

**Penjelasan komponen:**

1. **Sequence dibuat** - Object terpisah yang menghasilkan angka berurutan
2. **Kolom INTEGER NOT NULL** - Tipe data sebenarnya adalah integer
3. **DEFAULT nextval()** - Setiap insert otomatis memanggil fungsi untuk mendapat nilai berikutnya
4. **OWNED BY** - Sequence akan terhapus otomatis jika tabel/kolom dihapus

### b. Serial vs Identity (PostgreSQL 10+)

**Status serial saat ini:**

- ✅ Masih valid dan bisa digunakan
- ⚠️ **Tidak lagi recommended** untuk primary key sejak PostgreSQL 10
- 🆕 **Identity columns** adalah cara yang lebih baik (akan dibahas di video terpisah)

**Mengapa identity lebih baik:**

- Lebih sederhana
- Lebih sedikit caveat terkait permissions
- Syntax lebih standar SQL

**Catatan penting:**

```sql
-- Jika sudah pakai serial dari PostgreSQL 9, tidak perlu migrasi
-- Serial masih perfectly fine, hanya tidak recommended untuk project baru
```

### c. Varian Serial

PostgreSQL menyediakan beberapa varian serial berdasarkan ukuran:

| Tipe          | Underlying Type | Range                          | Catatan          |
| ------------- | --------------- | ------------------------------ | ---------------- |
| `SMALLSERIAL` | `SMALLINT`      | 1 to 32,767                    | Jarang digunakan |
| `SERIAL`      | `INTEGER`       | 1 to 2,147,483,647             | ~2.1 miliar      |
| `BIGSERIAL`   | `BIGINT`        | 1 to 9,223,372,036,854,775,807 | **Recommended**  |

**Contoh:**

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name TEXT
);
```

### d. Rekomendasi Krusial: Gunakan BIGSERIAL untuk Primary Keys

**⚠️ SANGAT PENTING:** Ini adalah **satu-satunya kasus** di mana disarankan menggunakan tipe data yang "terlalu besar".

**Prinsip umum di course ini:**

> "Pilih tipe data terkecil yang sesuai dengan data terbesar"

**KECUALI untuk primary keys:**

> "Berikan ruang sebesar mungkin untuk growth - gunakan BIGSERIAL"

**Alasan:**

- Running out of primary key space adalah **disaster**
- Sangat sulit dan menyakitkan untuk di-fix
- Butuh downtime dan migration yang kompleks

**Real-world case study - Basecamp:**

```sql
-- Basecamp menggunakan INTEGER untuk primary key
-- Mereka pikir 2.1 miliar cukup
-- TERNYATA TIDAK CUKUP
-- Mereka kehabisan primary key space dan mengalami masalah besar
```

**Best practice:**

```sql
-- ❌ JANGAN
CREATE TABLE users (
    id SERIAL PRIMARY KEY  -- Hanya 2.1 miliar
);

-- ✅ LAKUKAN
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY  -- 9+ quintillion
);
```

### e. Gaps dalam Sequence: Mengapa Terjadi?

**Gap dalam sequence adalah NORMAL dan AMAN** - bukan bug, tapi fitur!

**Skenario 1: Transaction Rollback**

```sql
-- Transaction 1 dimulai
BEGIN;
INSERT INTO users (name) VALUES ('Alice');
-- nextval() dipanggil, sequence sekarang di 5
-- Transaction 1 ROLLBACK
ROLLBACK;

-- Transaction 2 dimulai
BEGIN;
INSERT INTO users (name) VALUES ('Bob');
-- nextval() dipanggil, sequence sekarang di 6
COMMIT;

-- Hasil: id 1,2,3,4,6 (id 5 hilang selamanya)
```

**Mengapa ini desain yang bagus:**

```
Tanpa gaps (transaction-aware):
┌─────────────┬─────────────┐
│ Transaction 1│ Transaction 2│
├─────────────┼─────────────┤
│ Dapat ID 5  │ Dapat ID 5  │  ❌ DUPLICATE!
│ COMMIT      │ COMMIT      │  ❌ ERROR saat commit
└─────────────┴─────────────┘

Dengan gaps (tidak transaction-aware):
┌─────────────┬─────────────┐
│ Transaction 1│ Transaction 2│
├─────────────┼─────────────┤
│ Dapat ID 5  │ Dapat ID 6  │  ✅ Berbeda
│ ROLLBACK    │ COMMIT      │  ✅ Sukses
│ (5 hilang)  │             │  ✅ Aman
└─────────────┴─────────────┘
```

**Skenario 2: Concurrent Transactions**

Sequence increment **tidak di-rollback** karena:

- Mencegah duplicate key errors
- Memastikan concurrent transactions tidak bentrok
- Performance: tidak perlu locking yang kompleks

**Kapan gaps masalah:**

```sql
-- Jika business requirement HARUS strictly incrementing (tanpa gap)
-- Contoh: invoice number yang tidak boleh ada gap
-- Solusi: Harus implementasi custom logic, serial/sequence tidak cocok
```

### f. Serial sebagai Primary Key: Setup yang Benar

**Jika tetap menggunakan serial (legacy code, dll):**

```sql
-- ✅ BENAR: Deklarasi sebagai PRIMARY KEY
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name TEXT
);
```

**Mengapa PRIMARY KEY penting:**

1. Sequence sendiri tidak mencegah duplicate
2. User bisa manual insert value yang sama
3. User bisa reset sequence
4. PRIMARY KEY constraint memastikan uniqueness

**Contoh masalah tanpa PRIMARY KEY:**

```sql
-- Tanpa PRIMARY KEY constraint
CREATE TABLE users (
    id BIGSERIAL,
    name TEXT
);

-- Sequence di-reset (jangan lakukan ini!)
ALTER SEQUENCE users_id_seq RESTART WITH 1;

-- Manual insert
INSERT INTO users (id, name) VALUES (5, 'Alice');
INSERT INTO users (id, name) VALUES (5, 'Bob');  -- ✅ Sukses (SEHARUSNYA ERROR!)
```

**Rekomendasi untuk codebase lama:**

- Jika sudah pakai serial dari PostgreSQL 9: **tidak perlu migrasi**
- Tetap aman selama dideklarasi sebagai PRIMARY KEY
- Untuk table baru: gunakan identity columns (video terpisah)

### g. Use Case Lain: Order Number Generator

Serial tidak hanya untuk primary key - bisa untuk kolom lain yang butuh auto-increment.

**Contoh: Sistem order number:**

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,           -- Primary key internal
    order_number SERIAL,                 -- Order number untuk customer
    customer_name TEXT,
    total NUMERIC(10,2)
);
```

**Insert tanpa specify ID atau order_number:**

```sql
INSERT INTO orders (customer_name, total)
VALUES ('Alice', 150.00);

-- Hasil:
-- id: 1 (auto)
-- order_number: 1 (auto)
-- customer_name: Alice
-- total: 150.00
```

**Keuntungan punya kolom terpisah:**

- **Internal ID**: Bisa menggunakan UUID, big integer, atau format lain
- **Order Number**: Bisa custom format yang user-friendly untuk customer
- **Fleksibilitas**: Bisa set starting number sequence berbeda

**Custom starting number:**

```sql
-- Sequence bisa dimulai dari angka tertentu
-- Contoh: order number mulai dari 1000
ALTER SEQUENCE orders_order_number_seq RESTART WITH 1000;

-- Order berikutnya akan dapat order_number: 1000, 1001, 1002, ...
```

**Alternative dengan generated column:**

```sql
-- Video menyebut generated columns sebagai alternatif yang lebih baik
-- Akan dibahas di video terpisah
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number TEXT GENERATED ALWAYS AS ('ORD-' || id) STORED
);
```

### h. RETURNING Clause: Mendapatkan Auto-Generated Values

**Masalah umum:**

```sql
-- Insert data
INSERT INTO orders (customer_name, total)
VALUES ('Alice', 150.00);

-- Harus query lagi untuk dapat ID
SELECT id, order_number FROM orders
WHERE customer_name = 'Alice'
ORDER BY id DESC LIMIT 1;  -- ❌ Tidak reliable jika ada duplicate names
```

**Solusi: RETURNING clause**

```sql
-- ✅ Dapat nilai auto-generated dalam 1 query
INSERT INTO orders (customer_name, total)
VALUES ('Alice', 150.00)
RETURNING id, order_number;

-- Output:
-- id | order_number
-----------------
--  1 |      1001
```

**Kemampuan RETURNING:**

**1. Return kolom spesifik:**

```sql
INSERT INTO users (name, email)
VALUES ('Bob', 'bob@example.com')
RETURNING id;
-- Hanya dapat ID
```

**2. Return beberapa kolom:**

```sql
INSERT INTO orders (customer_name, total)
VALUES ('Charlie', 200.00)
RETURNING id, order_number;
-- Dapat ID dan order number
```

**3. Return seluruh row:**

```sql
INSERT INTO users (name, email)
VALUES ('Diana', 'diana@example.com')
RETURNING *;
-- Dapat semua kolom termasuk yang auto-generated
```

**Keuntungan:**

- **Efisiensi**: Satu round-trip ke database, bukan dua
- **Reliability**: Tidak perlu query lagi dengan WHERE yang mungkin tidak unique
- **Convenience**: Langsung dapat data yang dibutuhkan

**Use case dalam aplikasi:**

```javascript
// Contoh pseudo-code
const result = await db.query(
  `
    INSERT INTO orders (customer_name, total)
    VALUES ($1, $2)
    RETURNING id, order_number
`,
  ["Alice", 150.0]
);

// Langsung dapat ID dan order_number
console.log(result.rows[0].id); // 1
console.log(result.rows[0].order_number); // 1001
```

## 3. Hubungan Antar Konsep

**Flow pemahaman:**

1. **Serial adalah alias** → Bukan tipe data real, tapi shortcut untuk INTEGER + sequence + default
2. **Sequence adalah engine** → Object terpisah yang generate angka berurutan
3. **nextval() adalah mekanisme** → Fungsi yang dipanggil untuk mendapat nilai berikutnya
4. **Gaps adalah by design** → Sequence tidak transaction-aware untuk menghindari deadlock
5. **BIGSERIAL untuk primary key** → Satu-satunya kasus "over-provision" yang direkomendasikan
6. **PRIMARY KEY constraint wajib** → Sequence tidak guarantee uniqueness
7. **RETURNING clause** → Solusi elegant untuk mendapat auto-generated values

**Hierarki konsep:**

```
Serial Type
├── Expansion
│   ├── CREATE SEQUENCE
│   ├── INTEGER NOT NULL DEFAULT nextval()
│   └── ALTER SEQUENCE OWNED BY
├── Variants
│   ├── SMALLSERIAL (tidak recommended)
│   ├── SERIAL (okay, tapi...)
│   └── BIGSERIAL (✅ recommended untuk PK)
├── Behavior
│   ├── Auto-incrementing
│   ├── Not transaction-aware (gaps normal)
│   └── Requires PRIMARY KEY for uniqueness
├── Use Cases
│   ├── Primary Keys (gunakan identity di Postgres 10+)
│   ├── Order Numbers
│   └── Any auto-incrementing column
└── Best Practices
    ├── Gunakan BIGSERIAL untuk primary keys
    ├── Deklarasi sebagai PRIMARY KEY
    ├── Gunakan RETURNING untuk efficiency
    └── Migrate ke identity untuk project baru
```

**Timeline evolusi:**

```
PostgreSQL < 10: Serial adalah standard untuk primary key
         ↓
PostgreSQL 10+: Identity columns introduced (lebih baik)
         ↓
Sekarang: Serial masih valid, tapi identity recommended untuk project baru
```

## 4. Kesimpulan

Serial type adalah fitur PostgreSQL yang powerful untuk membuat kolom auto-incrementing, meskipun sebenarnya hanya alias untuk kombinasi INTEGER, sequence, dan default value.

**Key takeaways:**

**Tentang Serial:**

- Bukan tipe data real, tapi alias yang expand ke beberapa perintah
- Masih valid dan aman digunakan
- Sejak PostgreSQL 10, identity columns adalah cara yang lebih baik

**Best Practices KRUSIAL:**

- ✅ **Selalu gunakan BIGSERIAL untuk primary keys** (bukan SERIAL)
- ✅ Deklarasi sebagai PRIMARY KEY untuk ensure uniqueness
- ✅ Gunakan RETURNING clause untuk efficiency
- ⚠️ Gaps dalam sequence adalah NORMAL dan bukan masalah

**Real-world wisdom:**

- Kehabisan primary key space adalah disaster - don't let it happen
- Basecamp learn the hard way - gunakan BIGSERIAL
- Gap dalam sequence bukan bug, tapi design yang aman untuk concurrency

**Kapan menggunakan apa:**

- **Legacy codebase**: Tetap pakai serial (tidak perlu migrasi)
- **Project baru**: Gunakan identity columns (PostgreSQL 10+)
- **Non-primary-key use case**: Serial masih perfectly fine (contoh: order numbers)

**Next steps:**
Video selanjutnya akan membahas sequences secara lebih detail dan identity columns sebagai alternatif modern untuk primary keys.
