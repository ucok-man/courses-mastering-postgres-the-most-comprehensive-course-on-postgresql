# 📝 Generated Columns di PostgreSQL

## 1. Ringkasan Singkat

Generated columns adalah salah satu fitur favorit dalam database (termasuk PostgreSQL) yang berfungsi sebagai "escape hatch" atau solusi praktis untuk mengatasi berbagai masalah dalam schema database. Fitur ini memungkinkan kita membuat kolom yang nilainya dihitung secara otomatis berdasarkan kolom lain, mirip seperti formula di Excel atau Google Sheets. Generated columns sangat berguna baik untuk "menutupi" masalah dalam data model yang kurang rapi maupun untuk mempercantik query pada data model yang sudah bagus.

## 2. Konsep Utama

### a. Apa itu Generated Column?

Generated column adalah **kolom yang nilainya dihitung secara otomatis** berdasarkan nilai dari kolom lain dalam tabel yang sama. Perhitungan ini menggunakan ekspresi, fungsi, atau transformasi tertentu yang sudah didefinisikan saat tabel dibuat.

Kalau kamu familiar dengan Excel atau Google Sheets, konsep ini mirip seperti **cell yang berisi formula**—misalnya `=A1*2`. Nilainya tidak kamu isi manual, tapi dihasilkan dari perhitungan.

#### Karakteristik Generated Column

Berikut beberapa hal penting yang perlu dipahami:

- Generated column bekerja seperti **formula tetap di dalam database**
- Nilainya **tidak bisa diisi secara manual** saat `INSERT` atau diubah saat `UPDATE`
- Selalu **sinkron secara otomatis** dengan kolom yang menjadi sumbernya
- Cocok digunakan untuk menghindari duplikasi logika perhitungan di aplikasi

#### Contoh Sederhana

```sql
CREATE TABLE people (
  height_cm INTEGER,
  height_inches NUMERIC GENERATED ALWAYS AS (height_cm / 2.54) STORED
);
```

Mari kita bahas pelan-pelan:

- `height_cm` adalah kolom biasa yang diisi manual
- `height_inches` adalah generated column
- Ekspresi `(height_cm / 2.54)` digunakan untuk mengonversi tinggi dari sentimeter ke inci
- Keyword `GENERATED ALWAYS AS` berarti nilainya **selalu dihitung oleh database**
- Keyword `STORED` berarti hasil perhitungan disimpan secara fisik di tabel

#### Bagaimana Cara Kerjanya?

Misalnya kamu memasukkan data seperti ini:

```sql
INSERT INTO people (height_cm) VALUES (170);
```

Database akan otomatis menghitung:

- `height_inches = 170 / 2.54 ≈ 66.93`

Jadi ketika kamu query:

```sql
SELECT * FROM people;
```

Hasilnya kira-kira:

```
height_cm | height_inches
----------|---------------
170       | 66.93
```

#### Kenapa Ini Berguna?

Dengan generated column, kamu tidak perlu:

- Menghitung ulang nilai di aplikasi
- Khawatir data tidak konsisten
- Menyimpan data yang sebenarnya bisa dihitung

Semua logika perhitungan sudah “dipindahkan” ke database, sehingga lebih rapi, konsisten, dan minim error.

### b. Tipe Generated Column: Virtual vs Stored

Secara konsep umum, generated column dibagi menjadi dua tipe berdasarkan **kapan nilainya dihitung dan bagaimana penyimpanannya**.

#### 1. Virtual Generated Column

Virtual generated column adalah kolom yang:

- Nilainya **tidak disimpan secara fisik di database**
- Akan **dihitung setiap kali query dijalankan**
- Lebih hemat storage, karena tidak menyimpan hasil perhitungan

Namun, karena dihitung saat runtime, performanya bisa lebih lambat jika ekspresinya kompleks atau datanya besar.

#### 2. Stored Generated Column

Berbeda dengan virtual, stored generated column memiliki karakteristik:

- Nilainya **dihitung saat proses INSERT atau UPDATE**
- Hasilnya kemudian **disimpan secara fisik di dalam tabel**
- Saat di-query, database tinggal membaca nilai yang sudah ada (lebih cepat)

Konsekuensinya, stored column akan menggunakan ruang penyimpanan tambahan, tapi memberikan performa baca yang lebih baik.

#### Catatan Penting di PostgreSQL

Di PostgreSQL, kamu tidak punya pilihan antara virtual atau stored.

PostgreSQL **hanya mendukung generated column tipe stored**, sehingga:

- Keyword `STORED` **wajib ditulis**
- Tidak ada dukungan untuk virtual generated column

Ini penting untuk diingat, terutama kalau kamu pernah menggunakan database lain seperti MySQL yang mendukung kedua tipe.

#### Contoh Penggunaan

```sql
CREATE TABLE people (
  height_cm INTEGER,
  height_inches NUMERIC GENERATED ALWAYS AS (height_cm / 2.54) STORED
);
```

Penjelasan:

- `height_cm` diisi secara manual
- `height_inches` akan dihitung otomatis dari `height_cm`
- Keyword `STORED` memastikan hasil perhitungan disimpan di tabel

Sekarang kita masukkan data:

```sql
INSERT INTO people (height_cm) VALUES (200);
```

Saat perintah ini dijalankan, PostgreSQL langsung menghitung:

- `height_inches = 200 / 2.54 ≈ 78.74`

Nilai tersebut kemudian disimpan, bukan dihitung ulang setiap kali query.

Coba lihat hasilnya:

```sql
SELECT * FROM people;
```

Hasil yang akan kamu dapatkan:

```
height_cm | height_inches
----------|---------------
200       | 78.74
```

#### Intuisi Sederhana

Cara paling mudah memahami perbedaan ini:

- **Virtual** → seperti rumus yang dihitung setiap kali kamu buka file
- **Stored** → seperti hasil perhitungan yang sudah disimpan di dalam file

Karena PostgreSQL hanya mendukung stored, maka semua generated column di PostgreSQL bekerja dengan cara kedua: dihitung sekali saat data berubah, lalu disimpan untuk digunakan kembali.

### c. Use Case 1: Konversi Unit

Salah satu penggunaan paling praktis dari generated column adalah untuk **konversi unit**. Dengan pendekatan ini, kamu bisa menyimpan satu nilai utama, lalu secara otomatis mendapatkan versi lain dalam unit berbeda—tanpa khawatir datanya tidak konsisten.

#### Kenapa Ini Penting?

Dalam banyak kasus, data yang sama sering dibutuhkan dalam beberapa satuan. Misalnya:

- Tinggi badan dalam cm dan inci
- Berat dalam kg dan pound
- Jarak dalam kilometer dan mil

Kalau kamu menyimpan keduanya secara manual, ada risiko:

- Salah hitung
- Tidak konsisten (misalnya cm di-update, tapi inci lupa diubah)

Generated column menyelesaikan masalah ini dengan cara membuat satu nilai menjadi **sumber utama**, dan yang lain otomatis mengikuti.

#### Contoh Implementasi

```sql
CREATE TABLE people (
  id INTEGER PRIMARY KEY,
  height_cm INTEGER,
  height_inches NUMERIC GENERATED ALWAYS AS (height_cm / 2.54) STORED
);
```

Penjelasan:

- `height_cm` adalah nilai utama yang diinput oleh user
- `height_inches` adalah hasil konversi otomatis dari cm ke inci
- Rumus `(height_cm / 2.54)` digunakan untuk konversi
- Keyword `STORED` memastikan hasilnya disimpan di database

#### Menambahkan Data

```sql
INSERT INTO people (height_cm) VALUES (100);
INSERT INTO people (height_cm) VALUES (200);
```

Saat data dimasukkan, database langsung menghitung:

- 100 cm → `100 / 2.54 ≈ 39.37 inches`
- 200 cm → `200 / 2.54 ≈ 78.74 inches`

#### Melihat Hasil

```sql
SELECT * FROM people;
```

Hasilnya akan terlihat seperti ini:

```
id | height_cm | height_inches
---|-----------|---------------
1  | 100       | 39.37
2  | 200       | 78.74
```

Perhatikan bahwa kamu **tidak pernah mengisi `height_inches` secara manual**, tapi nilainya tetap tersedia dan akurat.

#### Apa yang Tidak Bisa Dilakukan?

Karena `height_inches` adalah generated column, kamu tidak boleh mengisinya secara langsung.

```sql
INSERT INTO people (height_cm, height_inches) VALUES (200, 78);
```

Perintah di atas akan menghasilkan error seperti:

```
Error: cannot insert into column "height_inches"
```

Ini terjadi karena database menjaga agar nilai tersebut **selalu berasal dari perhitungan**, bukan input manual.

#### Keuntungan Utama

Dengan pendekatan ini, kamu mendapatkan beberapa manfaat:

- Tidak ada risiko **data tidak sinkron**
- Tidak perlu menulis ulang logika konversi di aplikasi
- Data lebih **konsisten dan terjaga integritasnya**
- Query tetap cepat karena nilai sudah disimpan (stored)

Singkatnya, generated column membuat database “lebih pintar” dalam menangani data turunan seperti hasil konversi unit.

### d. Use Case 2: Ekstraksi Bagian dari String

Selain untuk perhitungan numerik seperti konversi unit, generated column juga sangat berguna untuk **memproses dan mengekstrak bagian tertentu dari string**. Salah satu contoh yang sering ditemui adalah mengambil domain dari sebuah alamat email.

#### Konsep Dasar

Misalnya kamu punya data email seperti:

```
aaron@tryhardstudios.com
```

Bagian yang sering dibutuhkan adalah domain-nya, yaitu:

```
tryhardstudios.com
```

Daripada setiap kali query harus memotong string ini secara manual, kamu bisa membuat generated column yang otomatis melakukan ekstraksi tersebut.

#### Contoh Implementasi

```sql id="j6k2pw"
CREATE TABLE users (
  id INTEGER PRIMARY KEY,
  email TEXT,
  email_domain TEXT GENERATED ALWAYS AS (
    SPLIT_PART(email, '@', 2)
  ) STORED
);
```

Penjelasan:

- `email` adalah kolom utama yang diisi oleh user
- `email_domain` adalah generated column
- Fungsi `SPLIT_PART(email, '@', 2)` digunakan untuk:
  - Memecah string berdasarkan karakter `@`
  - Mengambil bagian ke-2 (yaitu domain)

- Keyword `STORED` berarti hasilnya disimpan di tabel

#### Menambahkan Data

```sql id="0f94m5"
INSERT INTO users (email) VALUES ('aaron@tryhardstudios.com');
```

Saat data dimasukkan, PostgreSQL langsung memproses:

- Memecah email menjadi dua bagian: `aaron` dan `tryhardstudios.com`
- Menyimpan bagian domain ke kolom `email_domain`

#### Melihat Hasil

```sql id="0y4cnh"
SELECT * FROM users;
```

Hasilnya:

```id="i9bmqy"
id | email                      | email_domain
---|----------------------------|------------------------
1  | aaron@tryhardstudios.com   | tryhardstudios.com
```

Perhatikan bahwa `email_domain` terisi otomatis tanpa perlu kamu input.

#### Kenapa Pendekatan Ini Powerful?

Dengan adanya kolom `email_domain`, kamu bisa melakukan berbagai operasi dengan lebih efisien:

- **Filtering lebih cepat dan jelas**

  ```sql id="s5mtkk"
  SELECT * FROM users
  WHERE email_domain = 'gmail.com';
  ```

- **Grouping data berdasarkan domain**

  ```sql id="oaj93q"
  SELECT email_domain, COUNT(*)
  FROM users
  GROUP BY email_domain;
  ```

- **Membuat index untuk performa**

  Kamu bisa menambahkan index di `email_domain` agar query filtering dan grouping jadi lebih cepat.

#### Keuntungan Utama

Pendekatan ini memberikan beberapa manfaat penting:

- Tidak perlu menggunakan **pattern matching** seperti `LIKE '%@gmail.com'` yang lebih lambat
- Query jadi lebih sederhana dan mudah dibaca
- Performa meningkat, terutama jika data besar
- Konsistensi data terjaga karena ekstraksi dilakukan secara otomatis oleh database

Singkatnya, generated column membantu mengubah data mentah (seperti email) menjadi bentuk yang lebih siap digunakan untuk query dan analisis, tanpa membebani aplikasi.

### e. Kapan Menggunakan Generated Columns?

Generated column bukan hanya fitur “nice to have”, tapi bisa jadi solusi yang sangat praktis dalam berbagai situasi nyata di dunia pengembangan database. Intinya, fitur ini membantu kamu **menyederhanakan logika, menjaga konsistensi data, dan meningkatkan performa** tanpa perlu perubahan besar di sisi aplikasi.

Mari kita bahas kapan sebaiknya kamu mempertimbangkan untuk menggunakannya.

#### 1. Saat Requirements Bisnis Berubah

Dalam proyek nyata, kebutuhan bisnis hampir selalu berubah. Misalnya:

- Awalnya hanya menyimpan `price`
- Tiba-tiba butuh `price_with_tax`

Daripada mengubah banyak bagian di aplikasi, kamu bisa menambahkan generated column seperti:

```sql
price_with_tax NUMERIC GENERATED ALWAYS AS (price * 1.1) STORED
```

Dengan cara ini, kamu bisa menyesuaikan schema tanpa merusak sistem yang sudah berjalan.

#### 2. Saat Mewarisi Database Lama

Kadang kamu harus bekerja dengan database lama yang:

- Strukturnya tidak ideal
- Banyak data redundan
- Sulit diubah karena sudah dipakai banyak sistem

Generated column bisa jadi solusi “aman” untuk:

- Menambahkan struktur baru tanpa mengubah data lama
- Membuat representasi data yang lebih rapi

Misalnya, kamu bisa membuat kolom hasil ekstraksi atau perhitungan tanpa mengganggu kolom existing.

#### 3. Saat Deadline Ketat

Dalam kondisi deadline mepet, refactoring besar sering tidak memungkinkan.

Generated column bisa jadi “shortcut” yang efektif:

- Tidak perlu ubah logic di backend
- Tidak perlu migrasi data besar-besaran
- Langsung bisa digunakan di query

Ini sangat membantu untuk menambahkan fitur baru dengan risiko minimal.

#### 4. Untuk Mempermudah Query

Kalau kamu sering menulis query dengan:

- Perhitungan berulang
- Fungsi string yang sama
- Ekstraksi data yang kompleks

Generated column bisa menyederhanakan semuanya.

Contoh:

Daripada terus menulis:

```sql
SPLIT_PART(email, '@', 2)
```

Kamu cukup:

```sql
SELECT email_domain FROM users;
```

Query jadi lebih bersih, mudah dibaca, dan minim error.

#### 5. Untuk Optimasi Performa

Salah satu keuntungan besar generated column (khususnya yang bertipe STORED) adalah:

- Nilainya sudah dihitung dan disimpan
- Bisa dibuat **index**

Ini berarti:

- Query tidak perlu menghitung ulang setiap kali dijalankan
- Filtering dan grouping jadi lebih cepat

Contoh:

```sql
CREATE INDEX idx_email_domain ON users(email_domain);
```

Dengan index ini, query seperti:

```sql
SELECT * FROM users WHERE email_domain = 'gmail.com';
```

akan jauh lebih cepat dibandingkan menggunakan operasi string langsung di query.

#### Intuisi Sederhana

Gunakan generated column ketika:

- Kamu sering “mengolah ulang” data yang sama
- Ingin menjaga konsistensi tanpa bergantung pada aplikasi
- Butuh solusi cepat tanpa perubahan besar

Dengan kata lain, generated column membantu memindahkan sebagian “logika bisnis ringan” ke dalam database, sehingga sistem jadi lebih efisien dan terstruktur.

### f. Batasan (Restrictions) Generated Columns

Walaupun generated column sangat membantu, ada beberapa **batasan penting** yang perlu kamu pahami. Batasan ini ada untuk menjaga konsistensi data dan memastikan perhitungan tetap bisa diprediksi oleh database.

Mari kita bahas satu per satu dengan contoh agar lebih jelas.

#### 1. Hanya Boleh Mereferensi Row Saat Ini

Generated column hanya boleh menggunakan data dari **kolom lain dalam baris (row) yang sama**.

Artinya, kamu **tidak bisa**:

- Mengakses baris lain
- Mengakses tabel lain
- Menggunakan subquery

Contoh yang tidak diperbolehkan:

```sql
CREATE TABLE orders (
  id INTEGER,
  total NUMERIC GENERATED ALWAYS AS (
    (SELECT SUM(price) FROM order_items WHERE order_id = id)
  ) STORED
);
```

Kenapa ini tidak boleh?

Karena perhitungan `total` bergantung pada data di tabel lain (`order_items`), bukan hanya pada kolom dalam row yang sama. Generated column didesain untuk **perhitungan lokal**, bukan agregasi lintas tabel.

Jika kamu butuh kasus seperti ini, biasanya solusi yang lebih tepat adalah:

- Menggunakan query terpisah
- View
- Atau materialized view

#### 2. Fungsi Harus Deterministik (Tidak Boleh Volatile)

Generated column hanya boleh menggunakan fungsi yang **deterministik**, yaitu:

> Untuk input yang sama, hasilnya selalu sama

Sebaliknya, fungsi yang hasilnya bisa berubah-ubah (volatile) tidak diperbolehkan.

Contoh yang tidak boleh:

```sql
CREATE TABLE logs (
  id INTEGER,
  message TEXT,
  created_at TIMESTAMP GENERATED ALWAYS AS (NOW()) STORED,
  random_val NUMERIC GENERATED ALWAYS AS (RANDOM()) STORED
);
```

Masalahnya:

- `NOW()` akan selalu menghasilkan waktu saat ini (berubah-ubah)
- `RANDOM()` menghasilkan angka acak setiap kali dipanggil

Ini bertentangan dengan prinsip generated column yang harus **konsisten dan dapat diprediksi**.

Contoh yang benar:

```sql
CREATE TABLE products (
  id INTEGER,
  price NUMERIC,
  price_with_tax NUMERIC GENERATED ALWAYS AS (price * 1.10) STORED
);
```

Di sini:

- Input `price` sama → hasil `price_with_tax` juga selalu sama
- Aman dan valid untuk generated column

#### 3. Tidak Boleh Mereferensi Generated Column Lain

Generated column tidak boleh bergantung pada generated column lainnya dalam tabel yang sama.

Contoh yang tidak diperbolehkan:

```sql
CREATE TABLE measurements (
  meters NUMERIC,
  centimeters NUMERIC GENERATED ALWAYS AS (meters * 100) STORED,
  millimeters NUMERIC GENERATED ALWAYS AS (centimeters * 10) STORED
);
```

Kenapa ini error?

Karena `millimeters` mencoba menggunakan `centimeters`, yang juga merupakan generated column. PostgreSQL tidak mengizinkan chaining seperti ini.

#### Solusi yang Benar

Hitung langsung dari kolom sumber utama:

```sql
CREATE TABLE measurements (
  meters NUMERIC,
  centimeters NUMERIC GENERATED ALWAYS AS (meters * 100) STORED,
  millimeters NUMERIC GENERATED ALWAYS AS (meters * 1000) STORED
);
```

Dengan cara ini:

- Semua generated column hanya bergantung pada kolom asli (`meters`)
- Tidak ada dependency antar generated column

#### Intuisi Sederhana

Untuk memahami batasan ini, bayangkan generated column sebagai:

> “Rumus sederhana yang hanya boleh menggunakan data dari baris itu sendiri, dan hasilnya harus selalu sama.”

Jadi hindari:

- Logika lintas tabel
- Fungsi yang hasilnya berubah-ubah
- Ketergantungan antar generated column

Dengan memahami batasan ini, kamu bisa menggunakan generated column dengan lebih aman dan efektif tanpa menemui error yang membingungkan.

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
