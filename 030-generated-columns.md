# 📝 Generated Columns di PostgreSQL

## 1. Ringkasan Singkat

Generated columns adalah salah satu fitur favorit dalam database (termasuk PostgreSQL) yang berfungsi sebagai "escape hatch" atau solusi praktis untuk mengatasi berbagai masalah dalam schema database. Fitur ini memungkinkan kita membuat kolom yang nilainya dihitung secara otomatis berdasarkan kolom lain, mirip seperti formula di Excel atau Google Sheets. Generated columns sangat berguna baik untuk "menutupi" masalah dalam data model yang kurang rapi maupun untuk mempercantik query pada data model yang sudah bagus.

## 2. Konsep Utama

### a. Apa itu Generated Column?

Generated column adalah **kolom yang nilainya dihitung otomatis** berdasarkan kolom lain dalam tabel yang sama, dengan menerapkan transformasi atau fungsi tertentu.

**Karakteristik:**

- Seperti formula di Excel/Google Sheets yang mereferensi kolom lain
- Nilainya tidak bisa diisi secara manual (read-only untuk operasi INSERT/UPDATE)
- Selalu sinkron dengan kolom sumbernya

**Contoh sederhana:**

```sql
CREATE TABLE people (
  height_cm INTEGER,
  height_inches NUMERIC GENERATED ALWAYS AS (height_cm / 2.54) STORED
);
```

Pada contoh di atas, `height_inches` akan otomatis dihitung dari `height_cm`.

### b. Tipe Generated Column: Virtual vs Stored

Ada dua tipe generated column secara tradisional:

1. **Virtual**: Dihitung saat runtime (query dijalankan), tidak disimpan secara fisik
2. **Stored**: Dihitung saat INSERT/UPDATE, lalu disimpan secara fisik di storage

**⚠️ Penting:** PostgreSQL **hanya mendukung tipe STORED**. Keyword `STORED` wajib ditulis di akhir definisi generated column.

**Contoh:**

```sql
-- Di PostgreSQL, HARUS menggunakan STORED
CREATE TABLE people (
  height_cm INTEGER,
  height_inches NUMERIC GENERATED ALWAYS AS (height_cm / 2.54) STORED
);

INSERT INTO people (height_cm) VALUES (200);

SELECT * FROM people;
-- Hasilnya: height_cm = 200, height_inches = 78.74 (otomatis dihitung)
```

### c. Use Case 1: Konversi Unit

Generated columns sangat berguna untuk menyimpan data dalam unit berbeda tanpa risiko data tidak sinkron.

**Contoh:**

```sql
CREATE TABLE people (
  id INTEGER PRIMARY KEY,
  height_cm INTEGER,
  height_inches NUMERIC GENERATED ALWAYS AS (height_cm / 2.54) STORED
);

INSERT INTO people (height_cm) VALUES (100);
INSERT INTO people (height_cm) VALUES (200);

SELECT * FROM people;
-- Row 1: 100 cm = 39.37 inches
-- Row 2: 200 cm = 78.74 inches

-- ❌ Ini TIDAK BISA dilakukan (akan error):
INSERT INTO people (height_cm, height_inches) VALUES (200, 78);
-- Error: cannot insert into column "height_inches"
```

**Keuntungan:** Tidak ada risiko data height_cm dan height_inches tidak sinkron karena height_inches selalu dihitung otomatis.

### d. Use Case 2: Ekstraksi Bagian dari String

Generated columns sangat powerful untuk mengekstrak bagian tertentu dari string, misalnya domain dari email address.

**Contoh:**

```sql
CREATE TABLE users (
  id INTEGER PRIMARY KEY,
  email TEXT,
  email_domain TEXT GENERATED ALWAYS AS (
    SPLIT_PART(email, '@', 2)
  ) STORED
);

INSERT INTO users (email) VALUES ('aaron@tryhardstudios.com');

SELECT * FROM users;
-- email: aaron@tryhardstudios.com
-- email_domain: tryhardstudios.com (otomatis diekstrak)
```

**Keuntungan:**

- Bisa melakukan filtering berdasarkan domain: `WHERE email_domain = 'gmail.com'`
- Bisa melakukan grouping: `GROUP BY email_domain`
- Bisa membuat index pada `email_domain` untuk performa lebih baik
- Tidak perlu menggunakan pattern matching atau wildcard search yang lebih lambat

### e. Kapan Menggunakan Generated Columns?

Generated columns berguna dalam berbagai situasi:

1. **Saat requirements bisnis berubah** dan schema perlu disesuaikan
2. **Saat mewarisi database lama** yang struktur datanya kurang ideal
3. **Saat deadline ketat** dan butuh solusi cepat tanpa refactoring besar
4. **Untuk mempermudah query** yang sering melakukan kalkulasi atau ekstraksi data
5. **Untuk optimasi performa** dengan membuat index pada hasil kalkulasi

### f. Batasan (Restrictions) Generated Columns

Ada beberapa batasan penting yang perlu diperhatikan:

**1. Hanya boleh mereferensi row saat ini**

```sql
-- ❌ TIDAK BISA: mereferensi row lain, tabel lain, atau subquery
CREATE TABLE orders (
  id INTEGER,
  total NUMERIC GENERATED ALWAYS AS (
    (SELECT SUM(price) FROM order_items WHERE order_id = id)
  ) STORED
);
```

**2. Fungsi harus deterministik (tidak boleh volatile)**

```sql
-- ❌ TIDAK BISA: menggunakan fungsi yang hasilnya berubah-ubah
CREATE TABLE logs (
  id INTEGER,
  message TEXT,
  created_at TIMESTAMP GENERATED ALWAYS AS (NOW()) STORED,  -- ❌ NOW() tidak deterministik
  random_val NUMERIC GENERATED ALWAYS AS (RANDOM()) STORED  -- ❌ RANDOM() tidak deterministik
);

-- ✅ BOLEH: fungsi deterministik
CREATE TABLE products (
  id INTEGER,
  price NUMERIC,
  price_with_tax NUMERIC GENERATED ALWAYS AS (price * 1.10) STORED  -- ✅ Selalu sama untuk input yang sama
);
```

**3. Tidak boleh mereferensi generated column lain**

```sql
-- ❌ TIDAK BISA: generated column mereferensi generated column lain
CREATE TABLE measurements (
  meters NUMERIC,
  centimeters NUMERIC GENERATED ALWAYS AS (meters * 100) STORED,
  millimeters NUMERIC GENERATED ALWAYS AS (centimeters * 10) STORED  -- ❌ Error!
);

-- ✅ SOLUSI: Hitung langsung dari kolom asli
CREATE TABLE measurements (
  meters NUMERIC,
  centimeters NUMERIC GENERATED ALWAYS AS (meters * 100) STORED,
  millimeters NUMERIC GENERATED ALWAYS AS (meters * 1000) STORED  -- ✅ OK
);
```

## 3. Hubungan Antar Konsep

Generated columns bekerja sebagai **abstraksi** yang menyembunyikan kompleksitas kalkulasi atau transformasi data. Berikut alur pemikirannya:

1. **Problem**: Kita punya data dalam satu format, tapi sering perlu mengaksesnya dalam format lain
2. **Solution tradisional**: Kalkulasi di application code atau di setiap query (repetitif dan error-prone)
3. **Solution dengan generated columns**: Database yang menangani kalkulasi secara otomatis dan konsisten

**Flow diagram (konseptual):**

```
Data Asli (height_cm = 200)
         ↓
    GENERATED ALWAYS AS (height_cm / 2.54) STORED
         ↓
Data Turunan (height_inches = 78.74)
         ↓
    Otomatis ter-update saat height_cm berubah
```

**Keuntungan cascading:**

- Single source of truth (height_cm adalah sumber data)
- Tidak ada data duplikat yang bisa tidak sinkron
- Performa tetap baik karena STORED (sudah dihitung saat write)
- Query lebih sederhana karena tidak perlu kalkulasi berulang

**Contoh use case lanjutan:**

- Ekstraksi field dari JSON column
- Normalisasi text (lowercase, trim, etc.)
- Kalkulasi business logic (harga + pajak, diskon, dll.)
- Full-text search preparation (concatenate fields)

## 4. Kesimpulan

Generated columns adalah fitur powerful yang memberikan fleksibilitas dalam:

- ✅ Menjaga konsistensi data tanpa aplikasi logic
- ✅ Menyederhanakan query yang kompleks
- ✅ Memungkinkan indexing pada hasil kalkulasi
- ✅ "Menutupi" masalah pada schema atau data yang kurang ideal

**Prinsip penggunaan:**

- Gunakan untuk data yang bisa diturunkan (derived) dari kolom lain
- Pastikan fungsi deterministik (input sama = output sama)
- Gunakan `STORED` di PostgreSQL (wajib)
- Jangan ragu menggunakan fitur ini - ini bukan "hack", tapi solusi legitimate untuk banyak masalah real-world

Generated columns akan sering digunakan sepanjang course ini karena sangat praktis dan powerful baik untuk data model yang messy maupun yang pristine.
