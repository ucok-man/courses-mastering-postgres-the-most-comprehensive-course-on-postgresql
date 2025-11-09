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
4. **Performance hit** - Harus trim spasi saat operasi tertentu
5. **Perilaku tidak konsisten** - Tergantung operator dan collation:
   - Operator `LIKE`: Spasi dianggap signifikan
   - Operator `=`: Spasi dianggap tidak signifikan

**Contoh masalah:**

```sql
CREATE TABLE codes (
    code CHAR(5)
);

INSERT INTO codes VALUES ('ABC');  -- Berhasil! (padded jadi 'ABC  ')
-- Tapi kita mau enforce HARUS 5 karakter!

-- Perbandingan yang membingungkan:
SELECT 'ABC' = 'ABC  ';      -- TRUE (spasi diabaikan)
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
    content VARCHAR  -- Tanpa angka = unlimited
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

-- Bisa menyimpan data yang sangat besar
INSERT INTO articles VALUES (
    'My Article',
    'Konten yang sangat panjang... (bisa sampai gigabytes!)'
);
```

**Keuntungan TEXT:**

- ✅ Tidak ada batasan panjang
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

### e. TOAST - The Oversized-Attribute Storage Technique

TOAST adalah mekanisme internal PostgreSQL untuk menangani data text yang sangat besar.

**Cara kerja TOAST:**

1. Ketika text mencapai ukuran tertentu, PostgreSQL otomatis memindahkannya
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

## 3. Hubungan Antar Konsep

### Hirarki Rekomendasi (dari terbaik ke terburuk):

```
1. TEXT (dengan check constraint jika perlu)
   ↓
2. VARCHAR(n) (dengan n yang cukup besar)
   ↓
3. VARCHAR (tanpa limit, sama dengan TEXT)
   ↓
4. CHAR(n) ❌ JANGAN DIGUNAKAN!
```

### Proses Pengambilan Keputusan:

```
Apakah panjang HARUS tetap (misal: kode negara 'ID', 'US')?
├─ Ya → Gunakan TEXT/VARCHAR + CHECK constraint (dijelaskan video berikutnya)
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

1. **Default ke TEXT:**

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

2. **Kapan Menggunakan VARCHAR(n):**

   ```sql
   -- Gunakan hanya jika:
   -- a) Anda yang mengontrol data (bukan user input yang arbitrary)
   -- b) Batas sudah jelas dan tidak akan berubah

   CREATE TABLE settings (
       setting_key VARCHAR(50),    -- Kita yang define keys
       setting_value TEXT          -- Value bisa apa saja
   );
   ```

3. **Jangan Batasi Data Identitas Orang:**

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

4. **CHAR(n) Satu-satunya Use Case (Jarang!):**
   ```sql
   -- Jika Anda BENAR-BENAR butuh padding dengan spasi
   -- (sangat jarang, hampir tidak pernah)
   CREATE TABLE legacy_system (
       fixed_format CHAR(10)  -- Untuk kompatibilitas dengan sistem lama
   );
   -- Tapi bahkan untuk ini, lebih baik gunakan TEXT + CHECK!
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


-- ❌ JANGAN: Khawatir tentang TEXT "terlalu besar"
CREATE TABLE posts (
    content TEXT  -- "Takut nanti ada yang upload novel!"
);
-- TOAST akan handle ini otomatis!

-- ✅ LAKUKAN: Tambahkan constraint di aplikasi atau database
CREATE TABLE posts (
    content TEXT CHECK (length(content) <= 10000)
);
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

- **TEXT** = Kantong yang bisa mengembang tak terbatas
  - Isi sedikit → Kecil
  - Isi banyak → Besar, otomatis pindah ke gudang (TOAST)
  - ✅✅ Rekomendasi untuk kebanyakan kasus

### Mental Model untuk Memilih:

```
Pertanyaan: "Apakah saya tahu pasti berapa panjang maksimal data ini?"

Jika jawabnya:
- "Ya, dan itu identitas/kebutuhan user" → TEXT/VARCHAR(besar)
- "Ya, dan saya yang kontrol" → VARCHAR(n) OK
- "Tidak yakin" → TEXT
- "Harus panjang tetap" → TEXT + CHECK constraint (bukan CHAR!)
```

## 5. Kesimpulan

PostgreSQL memiliki tiga tipe data karakter yang **sebenarnya identik** dalam hal penyimpanan dan performa: CHAR(n), VARCHAR(n), dan TEXT. Ini berbeda dengan database lain di mana CHAR mungkin lebih cepat.

**Rekomendasi utama:**

1. **Jangan gunakan CHAR(n)** - Tidak ada keuntungan, banyak kerugian
2. **Gunakan TEXT sebagai default** - Fleksibel, performa bagus
3. **Gunakan VARCHAR(n) hanya jika:** Anda tahu pasti batasnya dan itu masuk akal
4. **Untuk data identitas orang:** Selalu beri ruang yang cukup (TEXT atau VARCHAR(255))
5. **Untuk data yang Anda kontrol:** VARCHAR(n) dengan batas yang reasonable

PostgreSQL punya TOAST yang otomatis menangani text yang sangat besar, jadi tidak perlu khawatir tentang performa untuk kolom TEXT yang besar.

**Key takeaway:** Di PostgreSQL, TEXT adalah pilihan paling aman dan fleksibel untuk hampir semua kasus. Tambahkan CHECK constraint di video berikutnya untuk enforcement yang proper, bukan mengandalkan CHAR(n) yang bermasalah.
