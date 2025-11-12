# 📝 Tipe Data Karakter di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tiga tipe data untuk menyimpan karakter di PostgreSQL: `CHAR` (fixed-length), `VARCHAR` (character varying), dan `TEXT`. Yang mengejutkan, ketiga tipe ini menggunakan struktur penyimpanan yang identik di balik layar, sehingga performa mereka sebenarnya sama. Video ini menjelaskan mengapa `CHAR` sebaiknya dihindari, kapan menggunakan `VARCHAR`, dan mengapa `TEXT` menjadi pilihan yang direkomendasikan dalam banyak kasus.

## 2. Konsep Utama

### a. Tiga Tipe Data Karakter

PostgreSQL memiliki tiga tipe data untuk karakter, namun semuanya **menggunakan struktur penyimpanan yang sama**:

1. **CHAR(n) / CHARACTER(n)** - Fixed-length (panjang tetap)
2. **VARCHAR(n) / CHARACTER VARYING(n)** - Variable-length dengan batas
3. **TEXT** - Variable-length tanpa batas

**Fakta mengejutkan:** Meskipun berbeda sintaksnya, ketiga tipe ini disimpan dengan cara yang identik di PostgreSQL!

**Perbedaan penting untuk CHAR dan VARCHAR tanpa parameter:**

```sql
-- PERHATIAN: Perilaku berbeda!
CREATE TABLE demo (
    col1 CHAR,       -- Sama dengan CHAR(1) - hanya 1 karakter!
    col2 VARCHAR,    -- Sama dengan TEXT - unlimited!
    col3 TEXT        -- Unlimited
);

INSERT INTO demo VALUES ('AB', 'AB', 'AB');
-- ERROR di kolom 'col1': value too long for type character(1)
-- OK untuk col2 dan col3
```

### b. CHAR(n) - Fixed-Length Character (❌ TIDAK DIREKOMENDASIKAN)

Tipe ini memiliki panjang tetap dan akan menambahkan spasi untuk mencapai panjang yang ditentukan.

```sql
-- Membuat tabel dengan CHAR
CREATE TABLE example (
    abbreviation CHAR(5)
);

-- Insert data yang lebih pendek dari 5 karakter
INSERT INTO example VALUES ('A');

-- Hasil: 'A    ' (dipadding dengan 4 spasi)
SELECT * FROM example;
SELECT length(abbreviation) FROM example;  -- 5 (bukan 1!)

-- Insert data yang lebih panjang dari 5 karakter
INSERT INTO example VALUES ('ABCDEF');  -- ERROR!
-- Error: value too long for type character(5)
```

**Masalah dengan CHAR(n):**

1. **Tidak enforce panjang minimum** - Bisa insert 1 karakter ke CHAR(5)
2. **Auto-padding dengan spasi** - Membuang ruang penyimpanan
3. **Tidak ada keuntungan performa** - Sama saja dengan VARCHAR/TEXT
4. **Performance overhead** - Peningkatan ruang penyimpanan dan siklus CPU tambahan untuk checking panjang
5. **Perilaku tidak konsisten** - Tergantung operator dan collation:
   - Operator `LIKE`: Spasi dianggap signifikan
   - Operator `=`: Spasi dianggap tidak signifikan (pada kebanyakan collation)

**Contoh masalah:**

```sql
CREATE TABLE codes (
    code CHAR(5)
);

INSERT INTO codes VALUES ('ABC');  -- Berhasil! (padded jadi 'ABC  ')
-- Tapi kita mau enforce HARUS 5 karakter!

-- Perbandingan yang membingungkan (perilaku bisa berbeda tergantung collation):
SELECT 'ABC' = 'ABC  ';      -- Umumnya TRUE (trailing spaces diabaikan)
SELECT 'ABC' LIKE 'ABC  ';   -- FALSE (spasi dianggap!)
```

### c. VARCHAR(n) / CHARACTER VARYING(n) - Variable-Length dengan Limit

Tipe ini memungkinkan penyimpanan karakter hingga batas maksimal yang ditentukan, **tanpa padding**.

```sql
-- Dengan limit
CREATE TABLE users (
    last_name VARCHAR(255)
);

-- Tanpa limit (sama dengan TEXT)
CREATE TABLE posts (
    content VARCHAR  -- Tanpa angka = unlimited, identik dengan TEXT
);

-- Insert data
INSERT INTO users VALUES ('Smith');       -- OK, disimpan sebagai 'Smith' (5 chars)
INSERT INTO users VALUES ('A');           -- OK, disimpan sebagai 'A' (1 char)
INSERT INTO users VALUES ('Long name...(256 chars)');  -- ERROR jika > 255!
```

**Karakteristik VARCHAR(n):**

- ✅ Tidak ada padding dengan spasi
- ✅ Ukuran penyimpanan sesuai panjang actual data
- ✅ Bisa membatasi panjang maksimal
- ⚠️ Pilih batas dengan hati-hati - sulit diubah nanti

**Peringatan penting tentang memilih batas:**

```sql
-- HATI-HATI dengan batas yang terlalu kecil!
CREATE TABLE people (
    last_name VARCHAR(30)  -- Mungkin terlalu kecil!
);

-- Nama ini valid tapi bisa ditolak:
-- "Wolfeschlegelsteinhausenbergerdorff" (35 chars)
-- "Hubert Blaine Wolfeschlegelsteinhausenbergerdorff" (sudah > 30!)

-- Lebih aman:
CREATE TABLE people (
    last_name VARCHAR(255)  -- Atau gunakan TEXT dengan constraint
);
```

### d. TEXT - Variable-Length Unlimited (✅ DIREKOMENDASIKAN)

Tipe ini tidak memiliki batas panjang dan identik dengan `VARCHAR` tanpa parameter.

```sql
-- TEXT dan VARCHAR tanpa limit adalah identik
CREATE TABLE articles (
    title TEXT,
    content VARCHAR  -- Sama persis dengan TEXT
);

-- Bisa menyimpan data yang sangat besar (hingga ~1GB)
INSERT INTO articles VALUES (
    'My Article',
    'Konten yang sangat panjang... (bisa sampai ~1GB!)'
);
```

**Keuntungan TEXT:**

- ✅ Tidak ada batasan panjang (maksimal ~1GB)
- ✅ Fleksibel untuk data yang tidak terprediksi
- ✅ Performa sama dengan VARCHAR
- ✅ Bisa tambahkan constraint kemudian jika perlu

**Catatan:** `TEXT` dan `VARCHAR` tanpa angka adalah **benar-benar identik** di PostgreSQL!

```sql
-- Kedua kolom ini 100% identik
CREATE TABLE demo (
    col1 TEXT,
    col2 VARCHAR
);
```

**Batasan maksimal:** String yang dapat disimpan adalah sekitar **1 GB**. Nilai maksimal yang diizinkan untuk n dalam deklarasi tipe data kurang dari itu.

### e. TOAST - The Oversized-Attribute Storage Technique

TOAST adalah mekanisme internal PostgreSQL untuk menangani data text yang sangat besar.

**Cara kerja TOAST:**

1. Ketika text mencapai ukuran tertentu (~2KB), PostgreSQL otomatis memindahkannya
2. Data dipindah ke tabel terpisah (invisible untuk user)
3. Row utama tetap compact di disk
4. Data besar bisa diakses saat dibutuhkan

```sql
-- Contoh: Menyimpan artikel yang sangat panjang
CREATE TABLE blog_posts (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT  -- Jika besar, auto-TOAST!
);

-- Insert artikel 1MB
INSERT INTO blog_posts (title, content)
VALUES ('Long Article', repeat('Lorem ipsum... ', 100000));

-- PostgreSQL otomatis:
-- 1. Menyimpan id dan title di tabel utama (compact!)
-- 2. Memindahkan content ke TOAST table
-- 3. Semua transparan, kita tidak perlu tahu!
```

**Analogi:**

- Tabel utama = Rumah kecil
- TOAST table = Gudang penyimpanan
- Barang besar otomatis dipindah ke gudang
- Kita masih bisa akses kapan saja

**Keuntungan:**

- Otomatis, tidak perlu konfigurasi
- Row tetap compact untuk operasi scan
- Performa optimal untuk query yang tidak butuh kolom besar
- MySQL punya fitur serupa (big text parking lot)

### f. Pertimbangan untuk Index

**PENTING:** PostgreSQL memiliki batas ukuran index **2712 bytes per row**. Ini perlu dipertimbangkan saat menggunakan TEXT untuk kolom yang di-index.

```sql
-- Masalah: Index limit 2712 bytes
CREATE TABLE urls (
    long_url TEXT
);
CREATE INDEX idx_url ON urls(long_url);
-- Bisa error jika data > 2712 bytes!
-- ERROR: index row size exceeds maximum

-- Solusi 1: Batasi dengan VARCHAR untuk kolom yang di-index
CREATE TABLE urls (
    long_url VARCHAR(500)  -- Aman untuk di-index
);
CREATE INDEX idx_url ON urls(long_url);

-- Solusi 2: TEXT dengan CHECK constraint
CREATE TABLE urls (
    long_url TEXT CHECK (length(long_url) <= 500)
);
CREATE INDEX idx_url ON urls(long_url);

-- Solusi 3: Partial/Expression index
CREATE TABLE urls (
    long_url TEXT  -- Tetap unlimited
);
CREATE INDEX idx_url ON urls(substring(long_url, 1, 255));

-- Solusi 4: Conditional partial index
CREATE INDEX idx_url ON urls(long_url)
WHERE length(long_url) <= 255;
```

**Kapan index limitation ini menjadi masalah:**

- URLs yang sangat panjang
- Hashed values atau tokens
- Serialized data (JSON, XML dalam TEXT)
- User-generated content yang di-index

**Best practice untuk indexed columns:**

```sql
-- Untuk kolom yang akan di-index: lebih aman pakai VARCHAR dengan limit
CREATE TABLE articles (
    title VARCHAR(255),  -- Di-index, batasi dengan aman
    slug VARCHAR(255),   -- Di-index, batasi dengan aman
    content TEXT,        -- Tidak di-index, bisa unlimited
    summary TEXT         -- Tidak di-index, bisa unlimited
);

CREATE INDEX idx_title ON articles(title);
CREATE INDEX idx_slug ON articles(slug);
-- Tidak perlu index untuk content dan summary
```

## 3. Hubungan Antar Konsep

### Hirarki Rekomendasi (dari terbaik ke terburuk):

```
1. TEXT (dengan check constraint jika perlu)
   ↓
2. VARCHAR(n) (dengan n yang cukup besar, terutama untuk indexed columns)
   ↓
3. VARCHAR (tanpa limit, sama dengan TEXT)
   ↓
4. CHAR(n) ❌ JANGAN DIGUNAKAN!
```

### Proses Pengambilan Keputusan:

```
Apakah kolom akan di-index?
├─ Ya → Gunakan VARCHAR(n) dengan n yang reasonable (< 2712 bytes)
│        Atau TEXT + CHECK constraint
└─ Tidak
    └─ Apakah panjang HARUS tetap (misal: kode negara 'ID', 'US')?
        ├─ Ya → Gunakan TEXT/VARCHAR + CHECK constraint (bukan CHAR!)
        └─ Tidak
            └─ Apakah ada batas maksimal yang masuk akal?
                ├─ Ya, dan pasti tidak akan berubah
                │   → VARCHAR(n) dengan n yang cukup besar
                │   (Contoh: judul blog post max 75 char - kita yang kontrol)
                └─ Tidak, atau batasnya tidak pasti
                    → TEXT
                    (Contoh: nama orang - identitas mereka, kita tidak bisa batasi)
```

### Perbandingan dengan Database Lain:

| Database   | CHAR(n)          | VARCHAR(n)      | TEXT          |
| ---------- | ---------------- | --------------- | ------------- |
| PostgreSQL | ❌ Hindari       | ✅ OK           | ✅✅ Terbaik  |
| MySQL      | Bisa lebih cepat | Standard choice | Tipe terpisah |
| SQL Server | Bisa lebih cepat | Standard choice | Tipe terpisah |

**PostgreSQL berbeda!** - Semua tipe char disimpan sama, tidak seperti database lain.

## 4. Catatan Tambahan / Insight

### Tips Praktis:

1. **Default ke TEXT (untuk kolom tanpa index):**

   ```sql
   -- Pendekatan yang aman dan fleksibel
   CREATE TABLE products (
       name TEXT,
       description TEXT,
       notes TEXT
   );

   -- Tambahkan constraint kemudian jika diperlukan
   ALTER TABLE products
   ADD CONSTRAINT name_length
   CHECK (length(name) <= 100);
   ```

2. **Gunakan VARCHAR(n) untuk Indexed Columns:**

   ```sql
   -- Best practice untuk kolom yang di-index
   CREATE TABLE articles (
       title VARCHAR(255),      -- Di-index: batasi dengan aman
       slug VARCHAR(255),       -- Di-index: batasi dengan aman
       content TEXT,            -- Tidak di-index: unlimited OK
       author_name VARCHAR(255) -- Mungkin di-index: batasi dengan aman
   );

   CREATE INDEX idx_title ON articles(title);
   CREATE INDEX idx_slug ON articles(slug);
   CREATE INDEX idx_author ON articles(author_name);
   ```

3. **Kapan Menggunakan VARCHAR(n) vs TEXT:**

   ```sql
   -- Gunakan VARCHAR(n) jika:
   -- a) Kolom akan di-index
   -- b) Anda yang mengontrol data (bukan user input yang arbitrary)
   -- c) Batas sudah jelas dan tidak akan berubah

   CREATE TABLE settings (
       setting_key VARCHAR(50),    -- Kita yang define keys, akan di-index
       setting_value TEXT          -- Value bisa apa saja, tidak di-index
   );

   CREATE INDEX idx_key ON settings(setting_key);

   -- Gunakan TEXT jika:
   -- a) Kolom tidak di-index
   -- b) Data dari user dan tidak bisa diprediksi panjangnya
   -- c) Fleksibilitas lebih penting daripada enforcement
   ```

4. **Jangan Batasi Data Identitas Orang:**

   ```sql
   -- JANGAN:
   CREATE TABLE customers (
       first_name VARCHAR(30),   -- ❌ Terlalu pendek!
       last_name VARCHAR(30)     -- ❌ Terlalu pendek!
   );

   -- LAKUKAN:
   CREATE TABLE customers (
       first_name TEXT,          -- ✅ Atau VARCHAR(255)
       last_name TEXT            -- ✅ Atau VARCHAR(255)
   );
   ```

5. **CHAR(n) Satu-satunya Use Case (Sangat Jarang!):**

   ```sql
   -- Jika Anda BENAR-BENAR butuh padding dengan spasi
   -- untuk kompatibilitas dengan sistem legacy
   CREATE TABLE legacy_system (
       fixed_format CHAR(10)  -- Untuk kompatibilitas dengan sistem lama
   );

   -- Tapi bahkan untuk ini, lebih baik gunakan TEXT + CHECK + padding manual!
   CREATE TABLE legacy_system (
       fixed_format TEXT CHECK (length(fixed_format) = 10)
   );
   ```

### Kesalahan Umum yang Harus Dihindari:

```sql
-- ❌ JANGAN: Menggunakan CHAR untuk "efisiensi"
CREATE TABLE codes (
    country_code CHAR(2)  -- 'ID', 'US', etc
);
-- Masalah: Tidak enforce 2 char, bisa insert 'A' → 'A '

-- ✅ LAKUKAN: TEXT dengan constraint
CREATE TABLE codes (
    country_code TEXT CHECK (length(country_code) = 2)
);
-- Enforce HARUS 2 karakter, tidak ada padding


-- ❌ JANGAN: VARCHAR dengan batas terlalu kecil untuk data identitas
CREATE TABLE people (
    name VARCHAR(20)  -- Terlalu kecil!
);

-- ✅ LAKUKAN: TEXT atau VARCHAR dengan batas yang reasonable
CREATE TABLE people (
    name TEXT  -- Atau VARCHAR(255)
);


-- ❌ JANGAN: TEXT untuk kolom yang akan di-index tanpa constraint
CREATE TABLE urls (
    url TEXT  -- Bisa error saat indexing jika URL > 2712 bytes!
);
CREATE INDEX idx_url ON urls(url);  -- ERROR possible!

-- ✅ LAKUKAN: VARCHAR atau TEXT dengan constraint untuk indexed columns
CREATE TABLE urls (
    url VARCHAR(500)  -- Aman untuk di-index
);
CREATE INDEX idx_url ON urls(url);

-- Atau:
CREATE TABLE urls (
    url TEXT CHECK (length(url) <= 500)
);
CREATE INDEX idx_url ON urls(url);


-- ❌ JANGAN: Khawatir tentang TEXT "terlalu besar" untuk non-indexed columns
CREATE TABLE posts (
    content TEXT  -- "Takut nanti ada yang upload novel!"
);
-- TOAST akan handle ini otomatis!

-- ✅ LAKUKAN: Tambahkan constraint jika memang perlu batas
CREATE TABLE posts (
    content TEXT CHECK (length(content) <= 10000)
);


-- ❌ JANGAN: Lupa bahwa CHAR tanpa parameter = CHAR(1)
CREATE TABLE test (
    code CHAR  -- Hanya 1 karakter!
);
INSERT INTO test VALUES ('AB');  -- ERROR!

-- ✅ LAKUKAN: Gunakan VARCHAR atau TEXT
CREATE TABLE test (
    code VARCHAR  -- Unlimited, sama dengan TEXT
);
-- Atau:
CREATE TABLE test (
    code TEXT
);
```

### Index Best Practices Summary:

```sql
-- Pattern 1: Kolom yang akan di-index
CREATE TABLE users (
    email VARCHAR(255),        -- Di-index: batasi
    username VARCHAR(50),      -- Di-index: batasi
    bio TEXT,                  -- Tidak di-index: unlimited OK
    full_name VARCHAR(255)     -- Mungkin di-index: batasi dengan aman
);

CREATE UNIQUE INDEX idx_email ON users(email);
CREATE UNIQUE INDEX idx_username ON users(username);
CREATE INDEX idx_name ON users(full_name);


-- Pattern 2: URL/Long strings yang perlu di-index
CREATE TABLE bookmarks (
    url TEXT,  -- Unlimited untuk storage
    title VARCHAR(255)
);

-- Index hanya sebagian URL
CREATE INDEX idx_url ON bookmarks(substring(url, 1, 255));

-- Atau gunakan hash index untuk exact match
CREATE INDEX idx_url_hash ON bookmarks USING hash(url);


-- Pattern 3: Conditional index untuk long text
CREATE TABLE documents (
    content TEXT
);

-- Index hanya untuk dokumen pendek
CREATE INDEX idx_short_content ON documents(content)
WHERE length(content) <= 255;
```

### Analogi Lengkap:

**Tipe Karakter seperti Wadah untuk Teks:**

- **CHAR(n)** = Kotak kaku ukuran tetap dengan bubble wrap

  - Isi "Hi" → Otomatis dikasih bubble wrap sampe penuh
  - Buang-buang tempat, susah handle
  - ❌ Jangan pakai!

- **VARCHAR(n)** = Kantong plastik dengan ukuran maksimal

  - Isi "Hi" → Hanya pakai ruang untuk "Hi"
  - Isi terlalu banyak → Ditolak
  - ✅ OK jika tahu pasti batasnya
  - ✅✅ Terbaik untuk kolom yang di-index

- **TEXT** = Kantong yang bisa mengembang tak terbatas
  - Isi sedikit → Kecil
  - Isi banyak → Besar, otomatis pindah ke gudang (TOAST)
  - ✅✅ Rekomendasi untuk kolom tanpa index
  - ⚠️ Hati-hati untuk kolom yang di-index (batasi dengan constraint)

**Index seperti Katalog:**

- Katalog hanya bisa muat kartu berukuran maksimal (2712 bytes)
- Jika data terlalu besar untuk kartu → tidak bisa masuk katalog (error)
- Solusi: potong data, atau batasi ukuran data yang masuk katalog

### Mental Model untuk Memilih:

```
Pertanyaan 1: "Apakah kolom ini akan di-index?"

Jika Ya:
  → Gunakan VARCHAR(n) dengan n yang reasonable (biasanya 255)
  → Atau TEXT + CHECK constraint untuk membatasi panjang

Jika Tidak:
  Pertanyaan 2: "Apakah saya tahu pasti berapa panjang maksimal data ini?"

  Jika jawabnya:
  - "Ya, dan itu identitas/kebutuhan user" → TEXT atau VARCHAR(besar)
  - "Ya, dan saya yang kontrol" → VARCHAR(n) OK
  - "Tidak yakin" → TEXT
  - "Harus panjang tetap" → TEXT + CHECK constraint (bukan CHAR!)
```

### Advanced Pattern: Hybrid Approach

```sql
-- Untuk kasus khusus: simpan full data + indexed portion
CREATE TABLE long_urls (
    id SERIAL PRIMARY KEY,
    full_url TEXT,                    -- Unlimited, untuk storage
    url_prefix VARCHAR(255),          -- Untuk indexing
    CHECK (url_prefix = substring(full_url, 1, 255))
);

CREATE INDEX idx_prefix ON long_urls(url_prefix);

-- Query menggunakan prefix untuk filtering cepat
SELECT * FROM long_urls
WHERE url_prefix = substring('https://example.com/very/long/path...', 1, 255);
```

## 5. Kesimpulan

PostgreSQL memiliki tiga tipe data karakter yang **sebenarnya identik** dalam hal penyimpanan dan performa: CHAR(n), VARCHAR(n), dan TEXT. Ini berbeda dengan database lain di mana CHAR mungkin lebih cepat.

**Rekomendasi utama:**

1. **Jangan gunakan CHAR(n)** - Tidak ada keuntungan, banyak kerugian
2. **Gunakan TEXT sebagai default untuk kolom tanpa index** - Fleksibel, performa bagus
3. **Gunakan VARCHAR(n) untuk kolom yang di-index** - Hindari index size limit (2712 bytes)
4. **Gunakan VARCHAR(n) hanya jika:** Anda tahu pasti batasnya dan itu masuk akal
5. **Untuk data identitas orang:** Selalu beri ruang yang cukup (TEXT atau VARCHAR(255))
6. **Untuk data yang Anda kontrol:** VARCHAR(n) dengan batas yang reasonable
7. **Ingat:** CHAR tanpa parameter = CHAR(1), VARCHAR tanpa parameter = TEXT

PostgreSQL punya TOAST yang otomatis menangani text yang sangat besar, jadi tidak perlu khawatir tentang performa untuk kolom TEXT yang besar **selama kolom tersebut tidak di-index**.

**Decision Tree Sederhana:**

```
Kolom akan di-index?
├─ Ya → VARCHAR(255) atau TEXT + CHECK(length <= 255)
└─ Tidak → TEXT (default), atau VARCHAR(n) jika perlu enforcement
```

**Key takeaway:** Di PostgreSQL, TEXT adalah pilihan paling aman dan fleksibel untuk hampir semua kasus **kecuali untuk kolom yang di-index**. Untuk kolom yang di-index, gunakan VARCHAR(n) dengan batas yang reasonable atau TEXT dengan CHECK constraint untuk menghindari index size limitation. Tambahkan CHECK constraint untuk enforcement yang proper, bukan mengandalkan CHAR(n) yang bermasalah.
