# 📝 Tipe Data Integer di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tipe data integer di PostgreSQL, fokus pada bagaimana memilih tipe integer yang tepat untuk data dalam bounded range (rentang terbatas). Pembahasan mencakup tiga tipe utama: `SMALLINT`, `INTEGER` (atau `INT`), dan `BIGINT`, serta bagaimana memilih yang paling sesuai berdasarkan prinsip **representative** (mencakup seluruh range data) dan **compact** (hemat space tanpa mengorbankan data).

## 2. Konsep Utama

### a. Prinsip Memilih Tipe Data Integer

Saat memilih tipe data integer, ada dua prinsip utama yang harus diperhatikan:

**1. Representative (Representatif)**

- Tipe data harus **mencakup seluruh range** data yang mungkin ada
- **JANGAN PERNAH** memotong atau memanipulasi data hanya untuk menghemat bytes
- Pertimbangkan **bounds of reality**, bukan hanya data saat ini

**2. Compact (Ringkas)**

- Jangan gunakan tipe data yang terlalu besar untuk range yang kecil
- Contoh: Jangan gunakan `BIGINT` untuk data 0-100

**Ilustrasi proses pemilihan:**

```
Langkah 1: Analisis data Anda
├── Periksa range data saat ini (contoh: 0-90 tahun)
└── Pertanyaan: Apakah ini bounds of reality?

Langkah 2: Tentukan bounds of reality
├── Data saat ini: 0-90 tahun
├── Reality check: Apakah ada orang hidup sampai 200+ tahun?
└── Pilihan reasonable: 0-1000 (untuk antisipasi masa depan)

Langkah 3: Match dengan tipe data PostgreSQL
└── Pilih tipe terkecil yang bisa menampung range tersebut
```

**Contoh kasus umur:**

```sql
-- ❌ BURUK: Menggunakan BIGINT untuk umur (overkill)
CREATE TABLE users (
    name TEXT,
    age BIGINT  -- Range: -9 quintillion sampai +9 quintillion
);

-- ✅ BAIK: Menggunakan SMALLINT (cukup untuk 0-32,767)
CREATE TABLE users (
    name TEXT,
    age SMALLINT  -- Range: -32,768 sampai 32,767
);
```

### b. Tiga Tipe Integer di PostgreSQL

PostgreSQL menyediakan tiga tipe integer dengan ukuran berbeda:

#### **1. SMALLINT (alias: INT2)**

**Spesifikasi:**

- **Ukuran:** 2 bytes
- **Range:** -32,768 sampai 32,767
- **Alias:** `INT2` (informative - "integer 2 bytes")

**Kapan menggunakan:**

- Data dengan range kecil (contoh: umur, persentase, rating)
- Nilai yang pasti tidak akan melebihi ±32 ribu

**Contoh:**

```sql
CREATE TABLE users (
    name TEXT,
    age INT2  -- Alias dari SMALLINT
);

-- Insert data normal
INSERT INTO users (name, age) VALUES
    ('Alice', 25),
    ('Bob', 67),
    ('Charlie', 0);  -- Bayi (0 tahun valid)

-- ❌ Insert data di luar range
INSERT INTO users (name, age) VALUES ('Robot', 40000);
-- ERROR: smallint out of range
```

**Catatan penting tentang error:**

- Error `out of range` memberikan **sedikit data integrity**
- Namun **bukan** solusi utama untuk business logic validation
- Contoh: 32,000 tahun juga tidak valid, tapi tidak error
- Gunakan **CHECK constraint** untuk validasi business logic yang tepat

#### **2. INTEGER (alias: INT, INT4)**

**Spesifikasi:**

- **Ukuran:** 4 bytes (2x lebih besar dari SMALLINT)
- **Range:** -2,147,483,648 sampai 2,147,483,647 (~±2.1 miliar)
- **Alias:** `INT` atau `INT4`

**Kapan menggunakan:**

- **Default pilihan** untuk kebanyakan kasus
- Data dengan range sedang sampai besar
- Banyak framework menggunakan ini sebagai default untuk migrations

**Contoh:**

```sql
CREATE TABLE warehouse_items (
    item_name TEXT,
    stock INT4  -- Bisa menampung hingga 2.1 miliar unit
);

INSERT INTO warehouse_items (item_name, stock) VALUES
    ('Laptop', 1500),
    ('Mouse', 25000),
    ('Monitor', 8500000);  -- Jutaan unit masih aman
```

**Rekomendasi penggunaan:**

- Untuk **kolom biasa**: `INTEGER` sudah cukup
- Untuk **Primary Key ID**: Lebih baik gunakan `BIGINT` (lihat di bawah)
- Jika yakin data sangat kecil: Gunakan `SMALLINT`

#### **3. BIGINT (alias: INT8)**

**Spesifikasi:**

- **Ukuran:** 8 bytes (4x lebih besar dari SMALLINT, 2x dari INTEGER)
- **Range:** -9,223,372,036,854,775,808 sampai 9,223,372,036,854,775,807
  - ~±9 **quintillion** (9 dengan 18 angka nol!)
- **Alias:** `INT8`

**Kapan menggunakan:**

- **Primary Key IDs** - SANGAT DIREKOMENDASIKAN
- Data yang bisa berkembang sangat besar (file size dalam bytes, timestamp UNIX, dll)
- Ketika tidak ingin khawatir kehabisan space

**Contoh:**

```sql
-- ✅ REKOMENDASI: Gunakan BIGINT untuk Primary Key
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,  -- Tidak akan kehabisan ID
    customer_id BIGINT,
    total_amount DECIMAL(10,2)
);

-- Contoh: File size dalam bytes
CREATE TABLE files (
    file_name TEXT,
    file_size_bytes BIGINT  -- File bisa sampai exabytes
);

INSERT INTO files (file_name, file_size_bytes) VALUES
    ('small.txt', 1024),
    ('large_video.mp4', 5368709120);  -- 5GB dalam bytes
```

**Perbandingan ukuran:**

```
SMALLINT: 2 bytes  [========]
INTEGER:  4 bytes  [================]
BIGINT:   8 bytes  [================================]

Pada 1 miliar rows:
- SMALLINT: 2 GB
- INTEGER:  4 GB
- BIGINT:   8 GB

Perbedaan bisa signifikan untuk table besar!
```

### c. Unsigned Integer di PostgreSQL

**Fakta penting:**

- PostgreSQL **TIDAK memiliki** tipe `UNSIGNED INTEGER`
- Berbeda dengan MySQL yang punya `UNSIGNED INT`
- Semua integer di PostgreSQL adalah **signed** (punya nilai negatif)

**Perbandingan dengan database lain:**

```sql
-- MySQL (memiliki UNSIGNED)
CREATE TABLE items (
    quantity INT UNSIGNED  -- Range: 0 sampai 4,294,967,295
);

-- PostgreSQL (TIDAK punya UNSIGNED)
CREATE TABLE items (
    quantity INTEGER  -- Range: -2.1 miliar sampai +2.1 miliar
);
```

**Solusi untuk enforce nilai positif:**

```sql
-- Gunakan CHECK constraint untuk memaksa nilai >= 0
CREATE TABLE products (
    product_name TEXT,
    quantity INTEGER CHECK (quantity >= 0)
);

-- Atau untuk umur (mulai dari 0)
CREATE TABLE users (
    name TEXT,
    age SMALLINT CHECK (age >= 0 AND age <= 150)
);

-- Test
INSERT INTO products (product_name, quantity) VALUES ('Laptop', -5);
-- ERROR: new row violates check constraint
```

**Catatan:**

- SQLite juga tidak punya unsigned integer (sama seperti PostgreSQL)
- MySQL user perlu adaptasi saat pindah ke PostgreSQL
- CHECK constraint lebih powerful untuk business logic daripada sekadar unsigned

### d. Penggunaan Framework untuk Migrations

**Perspektif tentang framework:**

Video memberikan pandangan seimbang tentang penggunaan framework:

✅ **Boleh menggunakan framework untuk migrations:**

- Tidak ada masalah menggunakan tools seperti Django ORM, Rails migrations, TypeORM, dll
- Anda **BUKAN** "developer palsu" jika tidak menulis SQL manual

⚠️ **TETAPI, Anda tetap bertanggung jawab:**

- Harus **tahu dan memahami** SQL yang di-generate oleh framework
- Harus bisa **review dan modify** migration files
- Harus memahami **implikasi** dari pilihan tipe data

**Contoh masalah umum framework:**

```javascript
// TypeORM - Default ke INTEGER untuk semua numeric
@Entity()
class User {
  @PrimaryGeneratedColumn()
  id: number; // Generated: INTEGER (bukan BIGINT!)

  @Column()
  age: number; // Generated: INTEGER (boros untuk umur!)
}
```

**Best practice:**

```javascript
// Specify tipe data secara eksplisit
@Entity()
class User {
  @PrimaryGeneratedColumn("bigint") // ✅ Explicit BIGINT untuk ID
  id: number;

  @Column({ type: "smallint" }) // ✅ Explicit SMALLINT untuk umur
  age: number;
}
```

### e. Primary Key: Selalu Gunakan BIGINT

**Rekomendasi kuat:**

```sql
-- ✅ BEST PRACTICE: Default ke BIGINT untuk Primary Key
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,  -- Auto-increment BIGINT
    username VARCHAR(50)
);

-- Alternatif manual
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name TEXT
);
```

**Alasan menggunakan BIGINT untuk PK:**

1. **Tidak ada downside signifikan:**

   - 8 bytes per row tidak mahal untuk PK
   - PK biasanya di-index, jadi performa tetap baik

2. **Menghindari disaster:**

   - INTEGER max = 2.1 miliar
   - Aplikasi yang populer bisa mencapai ini
   - **Migrasi dari INT ke BIGINT sangat menyakitkan** di production

3. **Peace of mind:**
   - 9 quintillion rows = praktis unlimited
   - Tidak perlu khawatir kehabisan ID

**Kisah horor (hypothetical):**

```sql
-- Tahun 2020: Aplikasi baru
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,  -- INTEGER (max 2.1 miliar)
    content TEXT
);

-- Tahun 2024: Aplikasi viral, 2 miliar posts
INSERT INTO posts (content) VALUES ('New post');
-- ERROR: integer out of range

-- Solusi: Migrasi ke BIGINT (sangat complex dan risky)
-- - Butuh downtime
-- - Harus update semua foreign keys
-- - Risk data corruption
```

## 3. Hubungan Antar Konsep

**Alur pemikiran dalam memilih tipe integer:**

```
1. Analisis Data
   ├── Apa bounds dari data saat ini?
   ├── Apa bounds of reality?
   └── Apakah data bisa berkembang drastis?

2. Evaluasi Tipe Data
   ├── SMALLINT: Untuk data kecil & stabil (umur, rating)
   ├── INTEGER: Default untuk kebanyakan kasus
   └── BIGINT: Untuk Primary Keys & data unlimited growth

3. Pertimbangan Tambahan
   ├── Apakah butuh unsigned? → Gunakan CHECK constraint
   ├── Business logic validation? → CHECK constraint / domain
   └── Framework default? → Review & customize bila perlu

4. Trade-off
   ├── Space vs Safety
   ├── SMALLINT hemat 6 bytes vs INTEGER (per row)
   └── Tapi jangan sacrifice data integrity!
```

**Hubungan dengan konsep lain:**

- **Check Constraints:** Melengkapi tipe data untuk business logic (dibahas video mendatang)
- **Domains:** Custom types untuk reusable constraints (dibahas video mendatang)
- **Indexing:** Tipe data lebih kecil = index lebih efisien
- **Table Compaction:** Column ordering untuk optimasi storage (dibahas jauh ke depan)

**Progression pemahaman:**

1. Pahami tipe data dasar (video ini)
2. Tambahkan constraints untuk validation
3. Optimize dengan indexing
4. Fine-tune dengan table structure optimization

## 4. Catatan Tambahan / Insight

### 💡 Tips Praktis

**1. Rule of Thumb untuk memilih tipe:**

```
Primary Key ID       → BIGINT (selalu!)
Umur, rating, skor   → SMALLINT
Jumlah, quantity     → INTEGER
File size (bytes)    → BIGINT
Years, dates range   → SMALLINT atau INTEGER
Counter unlimited    → BIGINT
```

**2. Jangan terlalu paranoid dengan size:**

```sql
-- ❌ Over-optimization yang tidak perlu
CREATE TABLE tiny_lookup (
    id SMALLINT,  -- Table cuma 10 rows, tapi pakai SMALLINT
    code CHAR(2)
);

-- ✅ Lebih baik: Gunakan INTEGER (consistency)
CREATE TABLE tiny_lookup (
    id INTEGER,   -- Consistency > micro-optimization
    code CHAR(2)
);
```

**3. Naming convention untuk clarity:**

```sql
-- INT2, INT4, INT8 lebih eksplisit tentang ukuran
CREATE TABLE metrics (
    small_value INT2,    -- Jelas: 2 bytes
    medium_value INT4,   -- Jelas: 4 bytes
    large_value INT8     -- Jelas: 8 bytes
);
```

### ⚠️ Kesalahan Umum

**1. Menggunakan INTEGER untuk Primary Key:**

```sql
-- ❌ RISIKO TINGGI
CREATE TABLE users (
    id SERIAL PRIMARY KEY  -- Max 2.1 miliar
);

-- ✅ AMAN
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY  -- Max 9 quintillion
);
```

**2. Mengandalkan "out of range" error untuk validation:**

```sql
-- ❌ BURUK: Mengandalkan SMALLINT untuk validasi umur
CREATE TABLE users (
    age SMALLINT  -- Max 32,767, tapi umur realistis max ~150
);
-- Problem: 32,000 akan diterima (invalid tapi tidak error)

-- ✅ BAIK: Explicit validation
CREATE TABLE users (
    age SMALLINT CHECK (age >= 0 AND age <= 150)
);
```

**3. Tidak review generated SQL dari framework:**

```python
# Django model
class Product(models.Model):
    quantity = models.IntegerField()  # Default: INTEGER

# Generated SQL: INTEGER (bisa jadi overkill atau kurang)
# Better: Review & specify bila perlu
# quantity = models.SmallIntegerField()  # Jika range kecil
```

### 🎯 Analogies & Mental Models

**Analogi wadah:**

```
SMALLINT  = Kotak kecil (muat 32 ribu item)
INTEGER   = Kardus sedang (muat 2 miliar item)
BIGINT    = Gudang besar (muat 9 quintillion item)

Umur (0-150)        → Kotak kecil (SMALLINT)
Stok produk (ribuan) → Kardus sedang (INTEGER)
Primary Key         → Gudang besar (BIGINT) - unlimited growth!
```

**Analogi check constraint vs unsigned:**

```
Unsigned (MySQL)     = Pagar built-in (hanya positif)
CHECK constraint (PG) = Custom security system (bisa atur apa saja)

PostgreSQL tidak punya pagar built-in,
tapi punya security system yang lebih flexible!
```

### 📊 Tabel Perbandingan Lengkap

```
╔══════════════╦═══════════╦════════════════════════╦═══════════════════╗
║ Tipe         ║ Bytes     ║ Range                  ║ Use Case          ║
╠══════════════╬═══════════╬════════════════════════╬═══════════════════╣
║ SMALLINT     ║ 2 bytes   ║ -32,768 to 32,767      ║ Age, small counts ║
║ (INT2)       ║           ║                        ║ ratings, scores   ║
╠══════════════╬═══════════╬════════════════════════╬═══════════════════╣
║ INTEGER      ║ 4 bytes   ║ -2.1B to 2.1B          ║ General purpose   ║
║ (INT, INT4)  ║           ║                        ║ quantities, years ║
╠══════════════╬═══════════╬════════════════════════╬═══════════════════╣
║ BIGINT       ║ 8 bytes   ║ -9 quintillion to      ║ Primary Keys,     ║
║ (INT8)       ║           ║ 9 quintillion          ║ file sizes, large ║
╚══════════════╩═══════════╩════════════════════════╩═══════════════════╝
```

### 🔧 Testing Your Understanding

**Quiz mental:**

1. Kolom untuk menyimpan rating film (1-5 bintang)?

   - **Jawaban:** SMALLINT (atau bahkan CHECK dengan INTEGER)

2. Kolom untuk User ID yang akan terus bertambah?

   - **Jawaban:** BIGINT (primary key rule)

3. Kolom untuk jumlah likes di post (bisa viral)?

   - **Jawaban:** INTEGER (atau BIGINT jika expect viral extreme)

4. Kolom untuk menyimpan tahun (2024)?
   - **Jawaban:** SMALLINT cukup (range ±32K)

## 5. Kesimpulan

Memilih tipe data integer yang tepat di PostgreSQL adalah tentang **balance** antara efisiensi storage dan safety data. Ada tiga pilihan utama:

- **SMALLINT (2 bytes):** Untuk data dengan range kecil dan stabil
- **INTEGER (4 bytes):** Default pilihan untuk kebanyakan kasus
- **BIGINT (8 bytes):** **Wajib** untuk Primary Key IDs, opsional untuk data unlimited growth

**Prinsip emas:**

1. **Analisis bounds of reality**, bukan hanya data saat ini
2. **Jangan sacrifice data integrity** demi hemat bytes
3. **Selalu gunakan BIGINT untuk Primary Keys**
4. **Gunakan CHECK constraint** untuk validation (bukan mengandalkan unsigned)
5. **Review generated SQL** dari framework - Anda tetap yang bertanggung jawab

PostgreSQL tidak memiliki unsigned integer seperti MySQL, tetapi CHECK constraints memberikan flexibility yang lebih powerful untuk business logic validation.

Pilihan tipe data yang tepat sejak awal akan mempermudah indexing, meningkatkan performa query, dan menghindari painful migration di masa depan. Ini adalah **investasi yang akan terbayar** seiring aplikasi berkembang.
