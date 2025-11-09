# 📝 Casting Types di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas berbagai cara melakukan casting (konversi tipe data) di PostgreSQL, serta beberapa fungsi utility yang berguna untuk debugging dan optimasi. Materi mencakup perbedaan antara syntax casting PostgreSQL dengan SQL standar, fungsi untuk memeriksa tipe data dan ukuran kolom, serta konsep decorated literal. Video juga membahas perdebatan tentang portabilitas database dan memberikan tips praktis untuk memilih tipe data yang tepat.

## 2. Konsep Utama

### a. Cara Melakukan Casting

Ada tiga cara utama untuk melakukan casting di PostgreSQL:

#### 1. PostgreSQL-specific syntax (double colon `::`)

Ini adalah cara yang paling umum dan ringkas di PostgreSQL.

```sql
SELECT 100::MONEY;
SELECT 100::INT8;
SELECT 100::INT4;
```

#### 2. SQL Standard syntax (CAST function)

Ini adalah cara standar SQL yang lebih portable.

```sql
SELECT CAST(100 AS MONEY);
SELECT CAST(100 AS INT8);
SELECT CAST(100 AS INT4);
```

#### 3. Decorated Literal

Cara yang kurang umum dengan menaruh tipe data di depan literal.

```sql
SELECT INTEGER '100';
-- Hasilnya adalah INT4
```

**Perbandingan:**

```sql
-- Ketiga cara ini menghasilkan output yang sama
SELECT 100::MONEY;
SELECT CAST(100 AS MONEY);
-- Keduanya menghasilkan: $100.00
```

**Catatan penting tentang Decorated Literal:**

- Tidak semua tipe data bisa menggunakan syntax ini
- Contohnya, `INT8 '100'` tidak akan bekerja, harus menggunakan `INTEGER '100'`
- Kurang dokumentasi dan jarang digunakan
- Memberikan hint ke PostgreSQL tentang tipe data literal

### b. Fungsi `pg_typeof()` - Memeriksa Tipe Data

Fungsi ini sangat berguna untuk debugging dan memastikan tipe data yang benar.

```sql
-- Membandingkan dua tipe data yang terlihat sama
SELECT 100::INT8, 100::INT4;
-- Output: 100, 100 (sulit dibedakan!)

-- Menggunakan pg_typeof untuk melihat perbedaannya
SELECT pg_typeof(100::INT8);  -- bigint
SELECT pg_typeof(100::INT4);  -- integer

-- Memastikan kedua kolom memiliki tipe yang sama
SELECT
    pg_typeof(100::INT8) AS tipe1,  -- bigint
    pg_typeof(100::INT8) AS tipe2;  -- bigint
```

**Kapan menggunakan `pg_typeof()`:**

- Saat debugging query yang kompleks
- Untuk memverifikasi output dari fungsi
- Saat tidak yakin tipe data apa yang dihasilkan suatu operasi

### c. Fungsi `pg_column_size()` - Memeriksa Ukuran Data

Fungsi ini menunjukkan berapa byte yang digunakan untuk menyimpan suatu nilai.

```sql
-- Memeriksa ukuran berbagai tipe integer
SELECT pg_column_size(100::INT2);  -- 2 bytes
SELECT pg_column_size(100::INT4);  -- 4 bytes
SELECT pg_column_size(100::INT8);  -- 8 bytes
```

**Pelajaran penting:** Nilai yang sama (100) menggunakan ruang yang berbeda tergantung tipe datanya!

#### Integer: Ukuran Fixed

```sql
-- Untuk integer, ukuran ditentukan oleh tipe kolom, bukan nilai
SELECT
    pg_column_size(100::INT2) AS small,      -- 2 bytes
    pg_column_size(100::INT4) AS medium,     -- 4 bytes
    pg_column_size(100::INT8) AS large;      -- 8 bytes
-- Semua menyimpan nilai 100, tapi ukuran berbeda!
```

#### Numeric: Ukuran Variable

```sql
-- Untuk numeric, ukuran berubah sesuai nilai yang disimpan
SELECT pg_column_size(100::NUMERIC);              -- 8 bytes
SELECT pg_column_size(10000.123456::NUMERIC);     -- 14 bytes
SELECT pg_column_size(10000.123456789012::NUMERIC); -- lebih besar lagi

-- Numeric bertambah ukurannya sesuai presisi yang dibutuhkan
```

### d. Perdebatan: Portabilitas vs Fitur Spesifik

Video ini memberikan opini tentang perdebatan portabilitas database:

**Argumen untuk SQL Standard (CAST):**

- Lebih portable ke database lain (MySQL, SQL Server, dll.)
- Berguna untuk library atau framework yang multi-database
- Standar industri

**Argumen untuk PostgreSQL Syntax (`::`):**

- Lebih ringkas dan mudah dibaca
- Jika sudah commit ke PostgreSQL, tidak perlu "nerf" implementasi
- Perpindahan database jarang terjadi dalam praktik

**Rekomendasi dari video:**

- Untuk aplikasi yang khusus PostgreSQL: gunakan `::`
- Untuk library/framework multi-database: gunakan `CAST`
- Jangan membatasi fitur hanya karena "mungkin" pindah database di masa depan

```sql
-- Pilihan 1: PostgreSQL style (lebih ringkas)
SELECT amount::NUMERIC(10,2) FROM transactions;

-- Pilihan 2: Standard SQL (lebih portable)
SELECT CAST(amount AS NUMERIC(10,2)) FROM transactions;
```

## 3. Hubungan Antar Konsep

### Workflow Optimasi Tipe Data:

1. **Pilih tipe data** yang sesuai dengan kebutuhan bisnis
2. **Gunakan `pg_column_size()`** untuk memeriksa ukuran actual
3. **Pilih tipe terkecil** yang bisa menampung data Anda
4. **Verifikasi dengan `pg_typeof()`** bahwa casting bekerja dengan benar

### Prinsip Penting:

```
Nilai yang sama + Tipe berbeda = Ukuran storage berbeda
```

**Untuk Integer:** Ukuran = Tipe kolom (fixed)
**Untuk Numeric:** Ukuran = Presisi data yang disimpan (variable)

### Contoh Praktis:

```sql
-- Scenario: Menyimpan umur seseorang (0-150)
-- Pilihan buruk:
CREATE TABLE users (age INT8);  -- 8 bytes per row

-- Pilihan baik:
CREATE TABLE users (age INT2);  -- 2 bytes per row
-- Hemat 6 bytes per row!

-- Verifikasi:
SELECT pg_column_size(25::INT8);  -- 8 bytes
SELECT pg_column_size(25::INT2);  -- 2 bytes
```

## 4. Catatan Tambahan / Insight

### Tips Praktis:

1. **Pilih Tipe Terkecil yang Cukup:**

   - Jangan default ke `BIGINT` untuk semua integer
   - Pertimbangkan range data yang mungkin
   - Gunakan `pg_column_size()` untuk membandingkan

2. **Untuk Tabel Besar:**

   - Perbedaan 2-4 bytes per kolom × jutaan rows = signifikan!
   - Contoh: 10 juta rows × 6 bytes saved = 60 MB saved per kolom

3. **Debugging Workflow:**

   ```sql
   -- Langkah 1: Lihat hasilnya
   SELECT my_function();

   -- Langkah 2: Cek tipe datanya
   SELECT pg_typeof(my_function());

   -- Langkah 3: Cek ukurannya
   SELECT pg_column_size(my_function());
   ```

4. **Decorated Literal:**
   - Jarang digunakan, tidak perlu dihafal
   - Jika menemukannya di code orang lain, Anda sekarang tahu apa itu
   - Lebih baik tetap gunakan casting biasa untuk konsistensi

### Analogi:

**Pemilihan Tipe Data seperti Memilih Kardus:**

- Data Anda = Barang yang mau dikirim
- Tipe Data = Ukuran kardus
- `INT2` = Kardus kecil (murah, efisien untuk barang kecil)
- `INT8` = Kardus besar (mahal, pemborosan untuk barang kecil)
- `NUMERIC` = Kardus yang bisa mengembang sesuai isi

Anda tidak akan pakai kardus besar untuk mengirim cincin, kan? Sama halnya, jangan pakai `BIGINT` untuk menyimpan umur!

### Peringatan:

```sql
-- JANGAN lakukan ini:
CREATE TABLE products (
    id BIGINT,           -- Overkill untuk ID < 2 miliar
    quantity BIGINT,     -- Overkill untuk stok < 32,767
    price NUMERIC        -- Tanpa constraint, bisa jadi sangat besar
);

-- LEBIH BAIK:
CREATE TABLE products (
    id INT,              -- Cukup sampai 2.1 miliar
    quantity INT2,       -- Cukup sampai 32,767
    price NUMERIC(10,2)  -- Max 99,999,999.99
);
```

## 5. Kesimpulan

Casting di PostgreSQL bisa dilakukan dengan tiga cara: syntax PostgreSQL (`::`, ringkas), SQL Standard (`CAST`, portable), dan decorated literal (jarang). Untuk debugging dan optimasi, PostgreSQL menyediakan fungsi utility:

- `pg_typeof()` untuk mengecek tipe data
- `pg_column_size()` untuk mengecek ukuran storage

**Prinsip utama:** Pilih tipe data terkecil yang bisa menampung data Anda. Ini penting karena tipe data menentukan ukuran storage (terutama untuk integer dengan ukuran fixed). Numeric memiliki ukuran variable yang tumbuh sesuai presisi data.

**Rekomendasi praktis:** Jika aplikasi Anda khusus menggunakan PostgreSQL, gunakan fitur-fitur PostgreSQL secara penuh (termasuk `::` syntax). Jangan membatasi diri hanya karena kemungkinan portabilitas di masa depan yang mungkin tidak pernah terjadi. Namun, untuk library atau framework, pertimbangkan SQL standard untuk kompatibilitas lebih luas.

**Key takeaway:** Kombinasikan `pg_typeof()` dan `pg_column_size()` untuk membuat keputusan yang tepat dalam schema design dan optimasi database.
