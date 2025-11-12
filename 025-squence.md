# 📝 Sequences di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas cara membuat dan memanipulasi **sequences** secara manual di PostgreSQL (tidak terikat dengan serial column). Sequences adalah object database yang menghasilkan angka berurutan dan dapat digunakan independen untuk berbagai keperluan. Video ini menjelaskan cara membuat sequence, fungsi-fungsi untuk memanipulasinya (`nextval`, `currval`, `setval`), dan perilaku sequence dalam konteks session yang berbeda.

## 2. Konsep Utama

### a. Membuat Sequence Manual

Berbeda dengan serial yang otomatis membuat sequence, kita bisa membuat sequence secara eksplisit untuk use case khusus.

**Syntax dasar:**

```sql
CREATE SEQUENCE sequence_name AS data_type;
```

**Contoh dengan opsi lengkap:**

```sql
CREATE SEQUENCE my_sequence AS BIGINT
    INCREMENT BY 1
    START WITH 1
    MINVALUE 1
    MAXVALUE 9223372036854775807
    CACHE 1;
```

**Parameter yang tersedia:**

| Parameter      | Default     | Deskripsi                           | Contoh Use Case                            |
| -------------- | ----------- | ----------------------------------- | ------------------------------------------ |
| `INCREMENT BY` | 1           | Berapa banyak increment setiap kali | `INCREMENT BY 10` untuk nomor kelipatan 10 |
| `START WITH`   | 1           | Nilai awal sequence                 | `START WITH 1000` untuk order number       |
| `MINVALUE`     | 1           | Nilai minimum                       | Untuk sequence yang bisa turun             |
| `MAXVALUE`     | max of type | Nilai maksimum                      | `MAXVALUE 100` untuk limit tertentu        |
| `CACHE`        | 1           | Berapa nilai di-cache di memory     | Performance optimization                   |

**Contoh praktis:**

```sql
-- Sequence untuk order number dimulai dari 1000
CREATE SEQUENCE order_number_seq AS INTEGER
    START WITH 1000
    INCREMENT BY 1;

-- Sequence yang increment by 10
CREATE SEQUENCE ticket_seq AS INTEGER
    START WITH 10
    INCREMENT BY 10;
-- Output: 10, 20, 30, 40, ...

-- Sequence dengan limit maksimum
CREATE SEQUENCE limited_seq AS INTEGER
    START WITH 1
    MAXVALUE 100;
-- Akan error setelah mencapai 100
```

### b. START WITH: Import Data dari Sistem Lain

Parameter `START WITH` sangat berguna saat migrasi atau import data.

**Use case: Import dari sistem legacy**

```sql
-- Skenario: Import data dari sistem lama
-- Sistem lama memiliki ID hingga 5432

-- 1. Import data dulu dengan ID existing
INSERT INTO users (id, name) VALUES
    (1, 'Old User 1'),
    (2, 'Old User 2'),
    -- ...
    (5432, 'Old User 5432');

-- 2. Buat sequence yang dimulai setelah data import
CREATE SEQUENCE users_id_seq AS BIGINT
    START WITH 5433;  -- Satu lebih tinggi dari max existing

-- 3. Set sebagai default untuk kolom
ALTER TABLE users
ALTER COLUMN id SET DEFAULT nextval('users_id_seq');

-- 4. Insert baru akan otomatis dapat ID 5433, 5434, ...
INSERT INTO users (name) VALUES ('New User');
-- ID: 5433
```

**Mengapa ini penting:**

- Menghindari duplicate key errors
- Seamless transition dari sistem lama
- Tidak perlu renumber data existing

### c. CACHE Parameter

**Apa itu CACHE:**

- Menentukan berapa banyak nilai sequence yang di-pre-allocate di memory
- Default: 1 (tidak ada caching)
- Trade-off antara performance dan potential gaps

**Cara kerja:**

```sql
CREATE SEQUENCE cached_seq AS INTEGER
    CACHE 20;

-- PostgreSQL akan:
-- 1. Pre-allocate nilai 1-20 di memory
-- 2. Ketika sampai 20, allocate lagi 21-40
-- 3. Ini mengurangi I/O ke disk
```

**Trade-off:**

```sql
-- Dengan CACHE 20
-- Jika server crash setelah menggunakan nilai 5
-- Nilai 6-20 akan hilang (gap)
-- Next value setelah restart: 21

-- Dengan CACHE 1 (default)
-- Lebih aman, tapi sedikit lebih lambat
-- Setiap nextval() perlu disk write
```

**Rekomendasi:**

- **Default (1)**: Untuk most cases, cukup aman
- **Lebih tinggi**: Untuk high-throughput systems yang okay dengan gaps

### d. nextval(): Mendapatkan Nilai Berikutnya

`nextval()` adalah fungsi utama untuk mengambil nilai dari sequence.

**Syntax:**

```sql
SELECT nextval('sequence_name');
```

**Contoh penggunaan:**

```sql
CREATE SEQUENCE demo_seq AS INTEGER;

-- Panggilan berturut-turut
SELECT nextval('demo_seq');  -- Output: 1
SELECT nextval('demo_seq');  -- Output: 2
SELECT nextval('demo_seq');  -- Output: 3
SELECT nextval('demo_seq');  -- Output: 4
```

**Karakteristik penting:**

- **Mengambil DAN increment**: Satu operasi yang atomic
- **Tidak bisa di-rollback**: Meskipun transaction gagal, nilai tetap terpakai
- **Thread-safe**: Aman untuk concurrent calls

**Dalam konteks INSERT:**

```sql
CREATE TABLE orders (
    id INTEGER DEFAULT nextval('order_seq'),
    order_date DATE
);

INSERT INTO orders (order_date) VALUES (CURRENT_DATE);
-- ID otomatis dari nextval('order_seq')
```

### e. currval(): Mendapatkan Nilai Terakhir dalam Session

`currval()` mengembalikan nilai **terakhir yang didapat dari nextval() dalam session yang sama**.

**⚠️ PENTING:** `currval()` adalah **session-specific**, bukan global sequence value!

**Contoh dasar:**

```sql
-- Session A
SELECT nextval('demo_seq');  -- Output: 10
SELECT currval('demo_seq');  -- Output: 10
SELECT currval('demo_seq');  -- Output: 10 (tetap sama)
```

**Perilaku multi-session:**

```sql
-- Session A
SELECT nextval('demo_seq');  -- Output: 28
SELECT currval('demo_seq');  -- Output: 28

-- Session B (berbeda)
SELECT nextval('demo_seq');  -- Output: 29
SELECT nextval('demo_seq');  -- Output: 30
-- ... hingga 44

-- Kembali ke Session A
SELECT currval('demo_seq');  -- Output: 28 (TETAP 28!)
-- Bukan 44, karena currval adalah per-session
```

**Visualisasi konsep:**

```
Global Sequence State: 1 → 2 → 3 → ... → 44

Session A:                   Session B:
├─ nextval() → 28           ├─ nextval() → 29
├─ currval() → 28           ├─ nextval() → 30
├─ currval() → 28           ├─ nextval() → 31
└─ currval() → 28           └─ ... → 44
   (tetap 28!)
```

**Use case praktis:**

```sql
BEGIN;

-- Ambil nilai untuk parent record
INSERT INTO orders (order_date)
VALUES (CURRENT_DATE)
RETURNING id;  -- Misalnya: 100

-- Gunakan nilai yang sama untuk child records
INSERT INTO order_items (order_id, product_id, quantity)
VALUES
    (currval('order_seq'), 1, 2),
    (currval('order_seq'), 2, 1),
    (currval('order_seq'), 3, 5);
-- Semua menggunakan order_id = 100

COMMIT;
```

**Error jika belum ada nextval:**

```sql
-- Session baru, belum pernah nextval
SELECT currval('demo_seq');
-- ERROR: currval of sequence "demo_seq" is not yet defined in this session

-- Harus nextval dulu
SELECT nextval('demo_seq');  -- Output: 50
SELECT currval('demo_seq');  -- Output: 50 ✅
```

**Kapan menggunakan currval:**

- Dalam transaction yang sama, butuh reference ke nilai yang sama
- Menghindari menyimpan nilai di variable aplikasi
- Memastikan referential integrity dalam multi-table insert

### f. setval(): Mereset atau Mengatur Sequence

`setval()` memungkinkan kita mengubah posisi sequence secara manual.

**Syntax:**

```sql
SELECT setval('sequence_name', new_value);
```

**Contoh penggunaan:**

**1. Reset ke awal:**

```sql
-- Sequence saat ini di 45
SELECT setval('demo_seq', 1);

SELECT nextval('demo_seq');  -- Output: 2
-- Mulai dari awal lagi
```

**2. Set ke nilai tinggi:**

```sql
-- Jump ke nilai tinggi
SELECT setval('demo_seq', 10000);

SELECT nextval('demo_seq');  -- Output: 10001
```

**3. Fix setelah manual insert:**

```sql
-- Skenario: Ada manual insert dengan ID tinggi
INSERT INTO users (id, name) VALUES (999, 'Special User');

-- Sequence masih di 100
-- Perlu set ke 999 agar tidak bentrok
SELECT setval('users_id_seq', 999);

-- Insert berikutnya aman
INSERT INTO users (name) VALUES ('Normal User');
-- ID: 1000
```

**⚠️ Bahaya setval():**

```sql
-- HATI-HATI: Reset ke nilai rendah
CREATE TABLE products (
    id INTEGER PRIMARY KEY DEFAULT nextval('product_seq'),
    name TEXT
);

-- Data existing: id 1-50
-- Sequence di 51

-- Reset (JANGAN LAKUKAN INI!)
SELECT setval('product_seq', 1);

-- Insert baru
INSERT INTO products (name) VALUES ('New Product');
-- ERROR: duplicate key value violates unique constraint
-- Karena id=2 sudah ada!
```

**Use case yang valid:**

1. **After data import**: Set sequence setelah import data
2. **Maintenance**: Memperbaiki sequence yang desync
3. **Testing**: Reset untuk test data
4. **Migration**: Sinkronisasi dengan sistem lain

**Best practice:**

```sql
-- Cari max ID yang ada
SELECT MAX(id) FROM users;  -- Misalnya: 5432

-- Set sequence sedikit lebih tinggi
SELECT setval('users_id_seq', 5432);

-- Atau gunakan query dynamic
SELECT setval('users_id_seq', (SELECT MAX(id) FROM users));
```

### g. Sequence dalam Multiple Sessions

Sequence adalah **shared object** yang bekerja lintas sessions.

**Demonstrasi:**

```sql
-- Terminal/Session 1
SELECT nextval('demo_seq');  -- 24
SELECT nextval('demo_seq');  -- 25

-- Terminal/Session 2 (concurrent)
SELECT nextval('demo_seq');  -- 26
SELECT nextval('demo_seq');  -- 27

-- Kembali ke Session 1
SELECT nextval('demo_seq');  -- 28 (lanjut dari Session 2)
```

**Karakteristik:**

- **Thread-safe**: Tidak ada race condition
- **Atomic**: Setiap nextval() adalah operasi atomic
- **No locks**: Tidak perlu explicit locking
- **Performance**: Sangat efficient untuk concurrent access

**Contrast dengan currval:**

```sql
-- Session A
SELECT nextval('demo_seq');  -- 10
-- (global sequence sekarang: 10)

-- Session B
SELECT nextval('demo_seq');  -- 11
SELECT nextval('demo_seq');  -- 12
-- (global sequence sekarang: 12)

-- Session A
SELECT currval('demo_seq');  -- Tetap 10!
-- currval() session-specific

SELECT nextval('demo_seq');  -- 13
-- nextval() ambil dari global state
```

### h. Use Cases untuk Manual Sequences

**1. Multiple tables sharing sequence:**

```sql
CREATE SEQUENCE global_id_seq AS BIGINT;

CREATE TABLE orders (
    id BIGINT DEFAULT nextval('global_id_seq'),
    -- ...
);

CREATE TABLE invoices (
    id BIGINT DEFAULT nextval('global_id_seq'),
    -- ...
);

-- Orders dan invoices share same sequence
-- Tidak akan ada ID conflict antar tables
```

**2. Custom number formatting:**

```sql
CREATE SEQUENCE invoice_seq START WITH 1000;

CREATE TABLE invoices (
    id BIGINT PRIMARY KEY,
    invoice_number TEXT GENERATED ALWAYS AS
        ('INV-' || TO_CHAR(id, 'FM000000')) STORED,
    -- INV-001000, INV-001001, ...
);
```

**3. Multiple sequences untuk partitioning:**

```sql
-- Sequence per region
CREATE SEQUENCE orders_us_seq;
CREATE SEQUENCE orders_eu_seq;
CREATE SEQUENCE orders_asia_seq;

-- Bisa track per-region ordering
```

**4. Temporary sequences untuk migrations:**

```sql
-- Sequence sementara untuk data migration
CREATE SEQUENCE migration_seq START WITH 1000000;

-- Migrate data dengan ID di range tertentu
-- Bisa di-drop setelah selesai
DROP SEQUENCE migration_seq;
```

## 3. Hubungan Antar Konsep

**Flow pemahaman:**

1. **Sequence adalah object independen** → Tidak harus terikat dengan serial column
2. **CREATE SEQUENCE dengan opsi** → Fleksibilitas full control atas behavior
3. **nextval() adalah operasi utama** → Atomic, thread-safe, tidak bisa rollback
4. **currval() adalah per-session** → Bukan global state, untuk reuse dalam transaction
5. **setval() untuk manual control** → Powerful tapi dangerous jika salah gunakan
6. **Multi-session behavior** → Sequence adalah shared resource yang aman

**Hierarki konsep:**

```
Sequences
├── Creation
│   ├── Data Type (INTEGER, BIGINT)
│   ├── INCREMENT BY (step size)
│   ├── START WITH (initial value)
│   ├── MIN/MAX VALUE (boundaries)
│   └── CACHE (performance tuning)
├── Operations
│   ├── nextval() - Get next value (atomic, global)
│   ├── currval() - Get last value (session-specific)
│   └── setval() - Reset/set value (dangerous)
├── Behavior
│   ├── Thread-safe
│   ├── Not transaction-aware
│   ├── Session-specific currval
│   └── Global shared state
└── Use Cases
    ├── Independent numbering
    ├── Shared across tables
    ├── Custom formats
    └── Migration support
```

**Relasi dengan konsep lain:**

```
Serial Type
    ↓ (creates automatically)
Sequence
    ↓ (used via)
nextval() / currval() / setval()
    ↓ (generates)
Auto-incrementing Values
```

**Perbedaan kunci:**
| Aspek | Serial | Manual Sequence |
|-------|--------|-----------------|
| Creation | Otomatis | Manual explicit |
| Ownership | Tied to column | Independent |
| Flexibility | Limited | Full control |
| Use case | Single column | Multiple uses |

## 4. Kesimpulan

Sequences adalah object database PostgreSQL yang powerful untuk menghasilkan angka berurutan secara atomic dan thread-safe. Berbeda dengan serial yang otomatis membuat sequence, kita bisa membuat sequence manual untuk use case yang lebih fleksibel.

**Key takeaways:**

**Tentang Creation:**

- ✅ Full control dengan `CREATE SEQUENCE`
- ✅ Banyak opsi: INCREMENT, START, MIN/MAX, CACHE
- ✅ `START WITH` berguna untuk data migration

**Tentang Operations:**

- `nextval()`: Get next value - **atomic, global, tidak bisa rollback**
- `currval()`: Get last value - **session-specific, bukan global**
- `setval()`: Manual reset - **powerful tapi dangerous**

**Best Practices:**

- 🎯 Gunakan CACHE default (1) kecuali perlu high-throughput
- ⚠️ **Hati-hati dengan setval()** - bisa menyebabkan duplicate key
- 📊 `currval()` perfect untuk multi-table insert dalam transaction
- 🔧 Manual sequence berguna untuk shared numbering antar tables

**Critical Understanding:**

- **nextval() adalah global**: Semua session lihat nilai terbaru
- **currval() adalah per-session**: Setiap session punya "memori" sendiri
- **Gaps adalah normal**: Transaction rollback tidak kembalikan nilai
- **Thread-safe**: Aman untuk concurrent access tanpa explicit locking

**When to use manual sequences:**

- Multiple tables sharing same sequence
- Custom numbering formats
- Data migration dengan ID control
- Temporary numbering systems

Sequences adalah fondasi dari auto-incrementing di PostgreSQL, dan memahami cara kerjanya penting untuk debugging dan optimization aplikasi database.
