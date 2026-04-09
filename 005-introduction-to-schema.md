# 📝 Pengenalan Schema di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas dua hal penting sebelum mempelajari berbagai tipe data di PostgreSQL: pertama, penjelasan tentang istilah "schema" yang memiliki makna berbeda dibanding database lain; kedua, pentingnya membangun struktur tabel dengan benar sejak awal untuk performa dan efisiensi database jangka panjang.

## 2. Konsep Utama

### a. Perbedaan Organisasi Database: SQLite, MySQL, dan PostgreSQL

Untuk memahami perbedaan SQLite, MySQL, dan PostgreSQL dengan baik, kita perlu melihat **bagaimana masing-masing sistem mengatur struktur databasenya**. Perbedaan ini bukan sekadar teknis, tetapi juga berpengaruh pada cara kita merancang aplikasi, mengelola data, dan mengatur akses.

#### SQLite: Struktur Paling Sederhana

SQLite menggunakan pendekatan yang sangat minimalis. Seluruh database disimpan dalam **satu file fisik** di sistem file.

Alurnya kurang lebih seperti ini:

- Satu file SQLite
- Di dalam file tersebut terdapat:

  - Tables
  - Data
  - Views
  - Triggers

Artinya, **satu file = satu database**. Tidak ada konsep server, tidak ada multiple database dalam satu instance. Karena itu, SQLite sangat cocok untuk aplikasi ringan, embedded system, atau aplikasi lokal yang tidak membutuhkan manajemen database kompleks.

#### MySQL: Server dengan Banyak Database

MySQL sudah menggunakan konsep **database server**. Dalam satu server MySQL, kita bisa memiliki **banyak database**.

Struktur organisasinya:

- MySQL Server

  - Database A
  - Database B
  - Database C

    - Tables
    - Data
    - Views
    - Triggers

Setiap database berdiri sendiri dan langsung berisi tabel-tabel. **Tidak ada lapisan tambahan di antara database dan tabel**. Model ini cukup sederhana dan umum dipakai di banyak aplikasi web.

#### PostgreSQL: Struktur Paling Lengkap dan Fleksibel

PostgreSQL memiliki struktur yang paling kaya dibandingkan SQLite dan MySQL. Di antara database dan tabel, PostgreSQL memperkenalkan konsep **schema**.

Struktur lengkapnya adalah:

- PostgreSQL Server / Cluster

  - Database

    - Schema

      - Tables
      - Data
      - Views
      - Triggers

Dengan kata lain, **satu database PostgreSQL bisa memiliki banyak schema**, dan setiap schema berisi objek-objek database.

#### Visualisasi Struktur PostgreSQL

Agar lebih mudah dibayangkan, berikut gambaran struktur PostgreSQL secara hierarkis:

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

Di sini terlihat jelas bahwa schema berfungsi sebagai **lapisan pengelompokan** di dalam satu database.

#### Catatan Penting tentang Schema di PostgreSQL

Ada beberapa hal penting yang perlu dipahami terkait schema:

- Schema dapat dianggap seperti **namespace** atau **folder logis** untuk mengelompokkan tabel dan objek lain.
- Schema **tidak bisa bersarang**. Artinya, tidak ada schema di dalam schema.
- Setiap database PostgreSQL **selalu memiliki schema default bernama `public`**.
  Jika kita membuat tabel tanpa menyebutkan schema secara eksplisit, tabel tersebut akan masuk ke schema `public`.

Dengan adanya schema, PostgreSQL menjadi sangat fleksibel untuk:

- Memisahkan domain data (misalnya `public`, `auth`, `analytics`)
- Menghindari bentrok nama tabel
- Mengatur hak akses dengan lebih rapi

Inilah alasan mengapa PostgreSQL sering dipilih untuk sistem berskala besar atau aplikasi yang membutuhkan pengelolaan data yang kompleks dan terstruktur.

### b. Dua Makna Kata "Schema"

Dalam dunia database—khususnya PostgreSQL—kata **"schema"** sering digunakan, tetapi maknanya bisa membingungkan jika tidak dilihat dari konteks yang tepat. Hal ini karena istilah **schema memiliki dua arti yang berbeda**, dan keduanya sama-sama valid serta sering dipakai dalam percakapan teknis sehari-hari.

#### Schema sebagai Namespace (Konsep PostgreSQL)

Makna pertama, dan yang paling spesifik di PostgreSQL, adalah **schema sebagai namespace**.
Dalam konteks ini, schema berfungsi sebagai **wadah atau container logis** untuk mengelompokkan objek database seperti table, view, function, dan trigger.

Saat seseorang bertanya:

- “Schema mana yang kamu query?”
- “Table ini ada di schema apa?”

Yang dimaksud adalah **lokasi logis table tersebut di dalam database**, misalnya `public`, `auth`, atau `analytics`.

Konsep ini mirip dengan:

- Folder dalam sistem file
- Namespace dalam bahasa pemrograman

Tujuannya adalah untuk:

- Menghindari bentrok nama table
- Mengelompokkan data berdasarkan domain atau fitur
- Mempermudah pengaturan hak akses

Sebagai contoh, kita bisa memiliki dua table dengan nama sama, tetapi berada di schema yang berbeda:

```sql
public.users
auth.users
```

Keduanya sah dan tidak saling bertabrakan karena berada di namespace yang berbeda.

#### Schema sebagai Struktur Table (Konsep Umum)

Makna kedua lebih umum dan tidak terbatas pada PostgreSQL. Di sini, **schema merujuk pada struktur atau definisi dari sebuah table**.

Jika ada pertanyaan seperti:

- “Apa schema dari table `users`?”

Yang dimaksud bukan lagi schema `public` atau `auth`, melainkan:

- Kolom apa saja yang dimiliki table tersebut
- Tipe data setiap kolom
- Constraint seperti `PRIMARY KEY`, `UNIQUE`, atau `NOT NULL`

Dengan kata lain, schema di sini berarti **blueprint atau desain table**.

#### Contoh Nyata dan Penjelasannya

Perhatikan contoh SQL berikut:

```sql
-- Membuat table di schema tertentu
CREATE TABLE public.users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    age SMALLINT
);
```

Dari contoh ini, terdapat dua penggunaan kata _schema_ dengan makna berbeda:

- `public.users`
  Kata `public` menunjukkan **schema sebagai namespace**, yaitu lokasi table di dalam database.

- Bagian dalam kurung `(id, username, age)`
  Ini adalah **schema sebagai struktur table**, yang menjelaskan:

  - Kolom `id` bertipe `SERIAL` dan menjadi `PRIMARY KEY`
  - Kolom `username` bertipe `VARCHAR(50)`
  - Kolom `age` bertipe `SMALLINT`

Memahami dua makna ini sangat penting agar tidak salah tafsir saat membaca dokumentasi, berdiskusi dengan tim, atau menulis desain database. Dalam praktik, konteks pembicaraan biasanya akan langsung menunjukkan schema yang mana yang sedang dimaksud.

### c. Prinsip Membangun Table Schema yang Baik

Saat merancang schema table, keputusan yang kita ambil akan berdampak langsung pada performa, konsistensi data, dan kemudahan pengelolaan aplikasi ke depannya. Karena itu, ada **tiga prinsip utama** yang sebaiknya selalu menjadi pegangan ketika menentukan struktur table dan tipe datanya.

#### Small (Kecil)

Prinsip _small_ berarti **memilih tipe data yang paling ringkas**, selama masih mampu merepresentasikan kebutuhan data dengan benar.

Semakin kecil tipe data:

- Semakin hemat ruang penyimpanan
- Semakin efisien akses data
- Semakin baik performa index dan query

Misalnya, untuk data yang hanya menyimpan angka kecil (seperti skor 0–100), menggunakan `BIGINT` jelas berlebihan. PostgreSQL menyediakan berbagai tipe numerik justru agar kita bisa memilih ukuran yang paling sesuai dengan kebutuhan nyata.

#### Simple (Sederhana)

Prinsip _simple_ menekankan bahwa **kesederhanaan adalah kekuatan**. Hindari desain schema yang rumit tanpa alasan yang jelas.

Kesalahan umum yang sering terjadi adalah:

- Menyimpan semua data sebagai `TEXT`
- Mengandalkan logika aplikasi untuk validasi, padahal database bisa melakukannya
- Menggunakan tipe data yang terlalu fleksibel padahal domain datanya jelas

Schema yang sederhana:

- Lebih mudah dipahami oleh developer lain
- Lebih mudah di-maintain
- Lebih kecil kemungkinan terjadi bug atau data tidak konsisten

#### Representative (Representatif)

Prinsip _representative_ berarti **tipe data harus mencerminkan sifat asli data** yang disimpan.

Contohnya:

- Uang sebaiknya disimpan sebagai tipe numerik (`NUMERIC`), bukan `TEXT`
- ID unik sebaiknya menggunakan `UUID`, bukan string biasa
- Nilai yang punya batasan domain sebaiknya diberi constraint yang sesuai

Dengan tipe data yang representatif, database dapat membantu menjaga validitas data sejak awal.

#### Contoh Implementasi dan Analisis

Perhatikan contoh schema berikut yang kurang baik:

```sql
-- ❌ BURUK: Menyimpan angka sebagai text
CREATE TABLE products (
    id TEXT,                    -- UUID disimpan sebagai text
    price TEXT,                 -- Angka disimpan sebagai text
    stock_quantity TEXT         -- Angka disimpan sebagai text
);
```

Pada desain ini:

- Database tidak tahu bahwa `price` dan `stock_quantity` adalah angka
- Sorting dan perhitungan numerik menjadi rumit
- Validasi sepenuhnya bergantung pada aplikasi

Sekarang bandingkan dengan desain yang lebih baik:

```sql
-- ✅ BAIK: Menggunakan tipe data yang tepat
CREATE TABLE products (
    id UUID PRIMARY KEY,               -- UUID menggunakan tipe UUID
    price NUMERIC(10,2),               -- Angka numeric untuk uang
    stock_quantity INTEGER             -- Integer untuk jumlah stok
);
```

Di sini:

- `UUID` secara eksplisit menunjukkan bahwa `id` adalah identifier unik
- `NUMERIC(10,2)` cocok untuk nilai uang karena presisi terjaga
- `INTEGER` sudah cukup untuk jumlah stok dalam kebanyakan kasus

Terakhir, contoh yang lebih matang dengan mempertimbangkan domain data:

```sql
-- ✅ LEBIH BAIK: Mempertimbangkan batasan domain
CREATE TABLE quiz_scores (
    user_id INTEGER,
    score SMALLINT CHECK (score >= 0 AND score <= 100)
    -- SMALLINT cukup untuk 0-100, tidak perlu INTEGER atau BIGINT
);
```

Pada contoh ini:

- `SMALLINT` dipilih karena rentang nilainya sudah lebih dari cukup
- Constraint `CHECK` memastikan skor tidak keluar dari batas logis
- Database ikut berperan aktif menjaga kualitas data

Dengan menerapkan prinsip **small, simple, dan representative**, schema table menjadi lebih efisien, aman, dan mudah dipahami—bukan hanya oleh database, tetapi juga oleh developer yang akan bekerja dengan schema tersebut di masa depan.

### d. Mengapa Memilih Tipe Data yang Tepat Itu Penting?

Memilih tipe data yang tepat dalam database sering dianggap sekadar urusan efisiensi penyimpanan. Padahal, dampaknya jauh lebih luas. Keputusan ini memengaruhi **validitas data, performa query, efisiensi index, hingga keterbacaan schema** oleh seluruh tim pengembang.

#### Enforcing Domain Logic (Menegakkan Logika Bisnis)

Dengan tipe data dan constraint yang tepat, database bisa ikut **menegakkan aturan bisnis**, bukan hanya aplikasi.

Contoh sederhana adalah nilai persentase yang secara logis hanya boleh berada di rentang 0 sampai 100:

```sql
-- Contoh: Persentase hanya bisa 0-100
CREATE TABLE ratings (
    rating SMALLINT CHECK (rating BETWEEN 0 AND 100)
);
```

Pada desain ini:

- `SMALLINT` membatasi ukuran data sejak level tipe
- `CHECK` constraint memastikan nilai tidak keluar dari domain yang valid

Jika ada attempt memasukkan nilai seperti `150` atau `-10`, PostgreSQL akan **menolak data tersebut secara otomatis**. Ini membuat data tetap konsisten meskipun ada bug di sisi aplikasi.

#### Indexing Efficiency (Efisiensi Indexing)

Ukuran tipe data berpengaruh langsung pada **ukuran dan kecepatan index**.

- Tipe data yang lebih kecil menghasilkan index yang lebih kecil
- Index yang lebih kecil lebih cepat dibaca dan lebih efisien di cache memory

Sebagai contoh, index pada kolom `SMALLINT` akan:

- Memakan ruang lebih sedikit
- Lebih cepat dibandingkan index pada `BIGINT`

Perbedaan ini mungkin tidak terasa pada table kecil, tetapi akan sangat signifikan pada table dengan jutaan baris.

#### Query Performance (Performa Query)

PostgreSQL memiliki **optimasi internal khusus untuk setiap tipe data**. Ketika kita menggunakan tipe data yang sesuai, query planner dapat memilih strategi eksekusi yang paling efisien.

Sebaliknya, jika angka disimpan sebagai `TEXT`:

- PostgreSQL tidak bisa menggunakan optimasi numerik
- Operasi perbandingan dan sorting menjadi lebih mahal
- Index numerik tidak bisa dimanfaatkan secara optimal

Dengan kata lain, tipe data yang salah bisa membuat database bekerja lebih keras dari yang seharusnya.

#### Code Clarity (Kejelasan Kode)

Schema database adalah bagian dari dokumentasi sistem. Tipe data yang tepat membantu developer lain **langsung memahami maksud data** tanpa perlu membaca kode aplikasi.

Contohnya:

- `age SMALLINT` langsung menunjukkan bahwa nilai ini adalah angka kecil dengan rentang terbatas
- `price NUMERIC(10,2)` langsung memberi sinyal bahwa ini adalah nilai uang

Kejelasan ini sangat penting dalam kerja tim dan maintenance jangka panjang.

#### Ilustrasi Perbandingan yang Lebih Nyata

Misalkan kita ingin menyimpan umur karyawan, dengan rentang realistis antara 18 hingga 70 tahun.

Desain yang kurang tepat:

```sql
-- ❌ Pemborosan: BIGINT bisa menyimpan hingga 9 quintillion
CREATE TABLE employees (
    age BIGINT  -- Overkill untuk data umur
);
```

Masalah pada desain ini:

- Kapasitas terlalu besar untuk kebutuhan sebenarnya
- Index menjadi lebih besar dan lebih lambat
- Tidak ada validasi domain

Bandingkan dengan desain yang lebih tepat:

```sql
-- ✅ Tepat: SMALLINT cukup (range: -32,768 to 32,767)
CREATE TABLE employees (
    age SMALLINT CHECK (age >= 18 AND age <= 100)
);
```

Keuntungan dari pendekatan ini:

- Ukuran data lebih kecil: `SMALLINT` hanya 2 byte, sedangkan `BIGINT` 8 byte (hemat sekitar 75%)
- Index lebih ringkas dan lebih cepat
- Validasi umur dilakukan langsung oleh database melalui `CHECK` constraint

Inilah alasan utama mengapa memilih tipe data yang tepat adalah investasi jangka panjang. Bukan hanya soal storage, tetapi juga tentang **kualitas data, performa sistem, dan kemudahan kolaborasi** dalam pengembangan aplikasi.

### e. Contoh Kasus: UUID vs TEXT

Untuk memahami pentingnya pemilihan tipe data yang tepat, mari kita lihat contoh yang sangat umum dalam praktik sehari-hari: **menyimpan UUID sebagai primary key**. Banyak sistem menggunakan UUID sebagai identifier unik, tetapi cara penyimpanannya sering kali kurang optimal.

#### Pendekatan yang Kurang Tepat: UUID sebagai TEXT

Berikut adalah contoh desain yang kurang baik:

```sql
-- ❌ BURUK: UUID disimpan sebagai TEXT
CREATE TABLE orders (
    order_id TEXT PRIMARY KEY  -- 36 karakter + overhead TEXT
);
```

Pada desain ini, UUID disimpan sebagai `TEXT`. Secara fungsional memang bekerja, tetapi ada beberapa konsekuensi penting:

- UUID versi string biasanya terdiri dari **36 karakter** (termasuk tanda `-`)
- Tipe `TEXT` memiliki overhead tambahan untuk menyimpan panjang data
- Database tidak tahu bahwa nilai tersebut adalah UUID, hanya dianggap sebagai string biasa

Akibatnya, database tidak bisa memberikan optimasi khusus untuk UUID, baik dari sisi validasi maupun performa.

#### Pendekatan yang Tepat: UUID Native PostgreSQL

Sekarang bandingkan dengan desain yang lebih baik:

```sql
-- ✅ BAIK: Menggunakan tipe UUID native
CREATE TABLE orders (
    order_id UUID PRIMARY KEY  -- 16 bytes, optimasi bawaan PostgreSQL
);
```

Dengan menggunakan tipe `UUID` bawaan PostgreSQL:

- Data disimpan dalam format biner **16 bytes**, jauh lebih ringkas
- PostgreSQL memahami bahwa kolom ini adalah UUID
- Banyak optimasi internal bisa langsung dimanfaatkan

#### Keuntungan Menggunakan Tipe UUID Native

Ada beberapa keuntungan nyata ketika menggunakan tipe `UUID` dibandingkan `TEXT`:

1. **Ukuran Lebih Kecil**
   UUID native hanya membutuhkan 16 bytes, sedangkan UUID dalam bentuk `TEXT` bisa memakan sekitar 36 karakter ditambah overhead. Ini berarti:

   - Lebih hemat storage
   - Index menjadi lebih kecil dan efisien

2. **Validasi Otomatis**
   PostgreSQL hanya akan menerima nilai yang benar-benar sesuai format UUID. Jika formatnya salah, data langsung ditolak. Ini mengurangi risiko data korup akibat input yang tidak valid.

3. **Fungsi Bawaan yang Kaya**
   PostgreSQL menyediakan fungsi-fungsi siap pakai seperti:

   - `gen_random_uuid()`
   - `uuid_generate_v4()`

   Fungsi ini memudahkan pembuatan UUID tanpa perlu logika tambahan di sisi aplikasi.

4. **Indexing Lebih Cepat**
   Karena ukuran data lebih kecil dan strukturnya konsisten, index pada kolom bertipe `UUID`:

   - Lebih ringkas
   - Lebih cepat di-scan
   - Lebih efisien saat digunakan dalam join atau pencarian berdasarkan primary key

#### Kesimpulan Praktis

Meskipun menyimpan UUID sebagai `TEXT` terlihat lebih fleksibel, pendekatan ini justru mengorbankan efisiensi dan kejelasan. Dengan menggunakan tipe `UUID` native, kita mendapatkan **validasi, performa, dan optimasi** secara gratis dari PostgreSQL.

Inilah contoh konkret bagaimana pemilihan tipe data yang tepat—bahkan untuk satu kolom—bisa memberikan dampak besar pada kualitas desain database secara keseluruhan.

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

## 5. Kesimpulan

PostgreSQL memiliki struktur hierarki yang lebih kompleks (Server → Database → **Schema** → Tables) dibanding database lain. Kata "schema" sendiri memiliki dua makna: sebagai namespace organizer dan sebagai definisi struktur table.

Saat membangun database, prioritaskan memilih tipe data yang **small, simple, dan representative**. Ini bukan sekadar tentang menghemat disk space, tetapi tentang:

- Menegakkan logika bisnis
- Meningkatkan efisiensi indexing
- Mengoptimalkan performa query
- Membuat kode lebih jelas dan maintainable

**Momen pembuatan schema adalah kesempatan pertama untuk "melakukannya dengan benar"**, yang akan memudahkan development, indexing, dan optimasi di masa depan. Meski schema bisa diubah nanti, memulai dengan fondasi yang solid akan menghemat banyak waktu dan effort dalam jangka panjang.
