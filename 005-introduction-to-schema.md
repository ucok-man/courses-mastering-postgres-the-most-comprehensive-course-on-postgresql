# 📝 Pengenalan Schema di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas dua hal penting sebelum mempelajari berbagai tipe data di PostgreSQL: pertama, penjelasan tentang istilah "schema" yang memiliki makna berbeda dibanding database lain; kedua, pentingnya membangun struktur tabel dengan benar sejak awal untuk performa dan efisiensi database jangka panjang.

## 2. Konsep Utama

### a. Perbedaan Organisasi Database: SQLite, MySQL, dan PostgreSQL

PostgreSQL memiliki **hierarki organisasi yang lebih kompleks** dibanding SQLite dan MySQL:

**SQLite:**

- File tunggal → Database → Tables, Data, Views, Triggers

**MySQL:**

- MySQL Server → Multiple Databases → Tables, Data, Views, Triggers

**PostgreSQL:**

- PostgreSQL Server/Cluster → Multiple Databases → **Multiple Schemas** → Tables, Data, Views, Triggers

**Visualisasi struktur PostgreSQL:**

```
PostgreSQL Cluster
└── Database 1
    ├── Schema 1 (contoh: public)
    │   ├── Table A
    │   ├── Table B
    │   └── View X
    ├── Schema 2 (contoh: analytics)
    │   ├── Table C
    │   └── Table D
    └── Schema 3
        └── ...
```

**Catatan penting:**

- Schema dalam PostgreSQL bisa dianggap seperti **namespace** atau **directory** (folder).
- Schema **tidak bisa nested** (tidak ada schema di dalam schema).
- Setiap database PostgreSQL memiliki schema default bernama **`public`**.

### b. Dua Makna Kata "Schema"

Kata "schema" dalam konteks PostgreSQL bisa berarti **dua hal berbeda**, tergantung konteks:

**1. Schema sebagai Namespace (konsep PostgreSQL):**

- Merujuk pada wadah/container yang mengelompokkan tables, views, dll.
- Contoh pertanyaan: _"Schema mana yang kamu query?"_ atau _"Table ini ada di schema apa?"_

**2. Schema sebagai Struktur Table (konsep umum):**

- Merujuk pada **struktur dan definisi** dari sebuah table.
- Contoh pertanyaan: _"Apa schema dari table `users`?"_ = "Apa kolom-kolomnya? Tipe datanya apa? Ada constraint apa?"

**Contoh:**

```sql
-- Membuat table di schema tertentu
CREATE TABLE public.users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    age SMALLINT
);

-- "public" = schema sebagai namespace
-- Struktur (id, username, age) = schema sebagai definisi table
```

### c. Prinsip Membangun Table Schema yang Baik

Ada **tiga prinsip utama** dalam merancang schema table:

**1. Small (Kecil)**

- Pilih tipe data yang paling ringkas namun tetap memadai.

**2. Simple (Sederhana)**

- Hindari kompleksitas yang tidak perlu.

**3. Representative (Representatif)**

- Tipe data harus **mencerminkan sifat data** yang sebenarnya.

**Contoh implementasi:**

```sql
-- ❌ BURUK: Menyimpan angka sebagai text
CREATE TABLE products (
    id TEXT,                    -- UUID disimpan sebagai text
    price TEXT,                 -- Angka disimpan sebagai text
    stock_quantity TEXT         -- Angka disimpan sebagai text
);

-- ✅ BAIK: Menggunakan tipe data yang tepat
CREATE TABLE products (
    id UUID PRIMARY KEY,               -- UUID menggunakan tipe UUID
    price DECIMAL(10,2),               -- Angka desimal untuk uang
    stock_quantity INTEGER             -- Integer untuk jumlah stok
);

-- ✅ LEBIH BAIK: Mempertimbangkan batasan domain
CREATE TABLE quiz_scores (
    user_id INTEGER,
    score SMALLINT CHECK (score >= 0 AND score <= 100)
    -- SMALLINT cukup untuk 0-100, tidak perlu INTEGER atau BIGINT
);
```

### d. Mengapa Memilih Tipe Data yang Tepat Itu Penting?

Memilih tipe data yang tepat **bukan hanya tentang menghemat disk space**, tetapi juga:

**1. Enforcing Domain Logic (Menegakkan Logika Bisnis)**

```sql
-- Contoh: Persentase hanya bisa 0-100
CREATE TABLE ratings (
    rating SMALLINT CHECK (rating BETWEEN 0 AND 100)
);
-- Database akan menolak nilai di luar range ini
```

**2. Indexing Efficiency (Efisiensi Indexing)**

- Kolom dengan tipe data yang lebih kecil → index yang lebih efisien
- Index pada `SMALLINT` lebih cepat daripada index pada `BIGINT`

**3. Query Performance (Performa Query)**

- PostgreSQL memiliki optimasi khusus untuk setiap tipe data
- Menyimpan angka di kolom `TEXT` membuat PostgreSQL tidak bisa menggunakan optimasi numerik

**4. Code Clarity (Kejelasan Kode)**

- Tipe data yang tepat membuat schema lebih mudah dipahami tim
- Developer lain langsung tahu jenis data yang diharapkan

**Ilustrasi perbandingan:**

```sql
-- Skenario: Menyimpan umur karyawan (range realistis: 18-70)

-- ❌ Pemborosan: BIGINT bisa menyimpan hingga 9 quintillion
CREATE TABLE employees (
    age BIGINT  -- Overkill untuk data umur
);

-- ✅ Tepat: SMALLINT cukup (range: -32,768 to 32,767)
CREATE TABLE employees (
    age SMALLINT CHECK (age >= 18 AND age <= 100)
);

-- Keuntungan:
-- - Ukuran: SMALLINT = 2 bytes, BIGINT = 8 bytes (hemat 75%)
-- - Index lebih kecil dan cepat
-- - Validasi built-in lewat CHECK constraint
```

### e. Contoh Kasus: UUID vs TEXT

```sql
-- ❌ BURUK: UUID disimpan sebagai TEXT
CREATE TABLE orders (
    order_id TEXT PRIMARY KEY  -- 36 karakter + overhead TEXT
);

-- ✅ BAIK: Menggunakan tipe UUID native
CREATE TABLE orders (
    order_id UUID PRIMARY KEY  -- 16 bytes, optimasi bawaan PostgreSQL
);

-- Keuntungan tipe UUID:
-- 1. Ukuran lebih kecil (16 bytes vs ~40 bytes)
-- 2. Validasi otomatis (hanya menerima format UUID valid)
-- 3. Fungsi built-in: gen_random_uuid(), uuid_generate_v4()
-- 4. Indexing lebih cepat
```

## 3. Hubungan Antar Konsep

**Alur pemahaman:**

1. **Memahami struktur PostgreSQL** (Database → Schema → Tables) penting agar tidak bingung saat membuat dan mengakses tables.

2. **Membedakan dua makna "schema"** membantu komunikasi yang lebih jelas dengan tim dan saat membaca dokumentasi.

3. **Memilih tipe data yang tepat** adalah fondasi dari performa database yang baik:

   - Schema yang baik → Indexing efisien → Query cepat
   - Tipe data representatif → Logika bisnis terjaga → Bug berkurang

4. **Prinsip "small, simple, representative"** berlaku di setiap tahap:
   - Small: Hemat resource, index efisien
   - Simple: Maintenance mudah
   - Representative: Menghindari bug, meningkatkan performa

## 4. Catatan Tambahan / Insight

### 💡 Tips Praktis:

1. **Jangan terlalu ekstrem dalam "menghemat space"**

   - ❌ Jangan sampai memotong atau memanipulasi data hanya demi hemat bytes
   - ✅ Fokus pada memilih tipe data yang **sesuai dengan domain bisnis**

2. **Pahami "universe of possibilities"**

   - Pelajari semua tipe data yang tersedia di PostgreSQL
   - Bandingkan dengan karakteristik data Anda
   - Pilih yang paling cocok

3. **Schema yang baik = investasi jangka panjang**
   - Lebih mudah diubah di awal daripada setelah production
   - Schema yang tepat memudahkan skalabilitas
   - Pengaruhnya terasa saat indexing dan optimasi query

### ⚠️ Kesalahan Umum:

1. **Menyimpan semua angka sebagai BIGINT "biar aman"**

   - Pemborosan space dan performa

2. **Menggunakan TEXT untuk semua hal**

   - Kehilangan validasi dan optimasi bawaan PostgreSQL

3. **Tidak mempertimbangkan batasan domain**
   - Contoh: Menyimpan nilai persentase dengan INTEGER tanpa CHECK constraint

### 🎯 Analogi:

Memilih tipe data seperti **memilih wadah untuk menyimpan barang**:

- Jangan simpan 1 permen dalam kardus besar (BIGINT untuk nilai 0-100)
- Jangan paksa simpan TV dalam kotak sepatu (TEXT untuk data kompleks)
- Pilih wadah yang **pas**: melindungi isinya, efisien, dan jelas fungsinya

## 5. Kesimpulan

PostgreSQL memiliki struktur hierarki yang lebih kompleks (Server → Database → **Schema** → Tables) dibanding database lain. Kata "schema" sendiri memiliki dua makna: sebagai namespace organizer dan sebagai definisi struktur table.

Saat membangun database, prioritaskan memilih tipe data yang **small, simple, dan representative**. Ini bukan sekadar tentang menghemat disk space, tetapi tentang:

- Menegakkan logika bisnis
- Meningkatkan efisiensi indexing
- Mengoptimalkan performa query
- Membuat kode lebih jelas dan maintainable

**Momen pembuatan schema adalah kesempatan pertama untuk "melakukannya dengan benar"**, yang akan memudahkan development, indexing, dan optimasi di masa depan. Meski schema bisa diubah nanti, memulai dengan fondasi yang solid akan menghemat banyak waktu dan effort dalam jangka panjang.
