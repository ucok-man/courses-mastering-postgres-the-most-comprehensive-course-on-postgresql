# 📝 Check Constraints di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang check constraints di PostgreSQL, sebuah mekanisme untuk menerapkan aturan validasi data langsung di level database. Check constraints memungkinkan kita untuk membatasi nilai yang bisa disimpan dalam kolom, seperti memastikan harga selalu positif atau panjang string harus tepat 5 karakter. Video juga membahas perbedaan antara column constraint dan table constraint, serta perdebatan tentang seberapa banyak business logic yang sebaiknya ada di database.

## 2. Konsep Utama

### a. Masalah yang Diselesaikan oleh Check Constraints

Check constraint hadir untuk menutup celah yang tidak bisa ditangani sepenuhnya oleh tipe data bawaan PostgreSQL. Meskipun PostgreSQL punya sistem tipe data yang kuat, ada beberapa aturan bisnis sederhana yang tidak bisa diekspresikan langsung hanya dengan memilih tipe data. Di sinilah check constraint berperan.

#### Keterbatasan Tipe Data Bawaan PostgreSQL

Beberapa database lain menyediakan variasi tipe data yang lebih spesifik untuk kasus tertentu. PostgreSQL memilih pendekatan yang lebih konsisten, tetapi konsekuensinya ada beberapa kebutuhan umum yang harus ditangani secara eksplisit oleh developer.

#### Problem 1: Integer Tidak Memiliki Versi Unsigned

Di beberapa database seperti MySQL, kita bisa menggunakan integer unsigned untuk memastikan nilai selalu non-negatif dan sekaligus memanfaatkan seluruh rentang bilangan positif.

```sql
-- Di database lain (MySQL, dll):
CREATE TABLE products (
    quantity UNSIGNED INTEGER  -- Range: 0 sampai 4.2 miliar
);
```

Di PostgreSQL, tipe `INTEGER` selalu bertanda (signed). Artinya, setengah dari rentang nilai disediakan untuk bilangan negatif.

```sql
-- Di PostgreSQL, INTEGER selalu signed:
CREATE TABLE products (
    quantity INTEGER  -- Range: -2.1 miliar sampai +2.1 miliar
);
-- Kita "kehilangan" setengah range untuk nilai negatif yang tidak terpakai!
```

Untuk kolom seperti `quantity`, nilai negatif sebenarnya tidak masuk akal. Namun tanpa aturan tambahan, PostgreSQL tetap mengizinkan nilai seperti `-10`. Masalah ini bukan soal teknis semata, tetapi soal validasi data sesuai domain bisnis.

#### Problem 2: CHAR(n) Tidak Memaksa Panjang Input Secara Logis

Sekilas, `CHAR(5)` terlihat seperti solusi tepat jika kita ingin menyimpan kode dengan panjang lima karakter.

```sql
CREATE TABLE codes (
    abbreviation CHAR(5)
);
```

Namun perilaku `CHAR(n)` di PostgreSQL adalah melakukan padding dengan spasi, bukan menolak input yang lebih pendek.

```sql
INSERT INTO codes VALUES ('A');  -- Berhasil! (disimpan sebagai 'A    ')
```

Secara teknis ini valid, tetapi secara logika sering kali tidak diinginkan. Jika aturan bisnis menyatakan bahwa kode _harus_ terdiri dari tepat lima karakter, maka padding otomatis justru menyembunyikan kesalahan input.

#### Solusi: Check Constraints

Check constraint memungkinkan kita mendefinisikan aturan validasi langsung di level database. Aturan ini akan selalu dievaluasi setiap kali data dimasukkan atau diubah.

Untuk kasus integer non-negatif, kita bisa memaksa nilainya selalu lebih besar atau sama dengan nol.

```sql
CREATE TABLE products (
    quantity INTEGER CHECK (quantity >= 0)
);
```

Dengan constraint ini, PostgreSQL akan menolak nilai negatif sejak awal, sehingga integritas data tetap terjaga tanpa perlu bergantung pada logika aplikasi.

Untuk kasus panjang string yang harus tepat lima karakter, pendekatan yang lebih tepat adalah menggunakan `TEXT` lalu menambahkan validasi eksplisit.

```sql
CREATE TABLE codes (
    abbreviation TEXT CHECK (length(abbreviation) = 5)
);
```

Sekarang, setiap input yang panjangnya kurang atau lebih dari lima karakter akan langsung ditolak oleh database. Aturan bisnis menjadi jelas, eksplisit, dan konsisten, serta tidak bergantung pada asumsi perilaku tipe data bawaan.

Dengan cara ini, check constraint berfungsi sebagai lapisan perlindungan tambahan yang memastikan data yang tersimpan benar-benar sesuai dengan makna dan kebutuhan domain aplikasi.

### b. Syntax Dasar Check Constraint

Check constraint bisa ditulis dengan beberapa gaya penulisan, tetapi secara konsep semuanya melakukan hal yang sama: memvalidasi data sebelum benar-benar disimpan ke dalam tabel. Perbedaan utamanya terletak pada seberapa eksplisit dan seberapa mudah constraint tersebut dipahami saat terjadi error atau saat maintenance.

#### 1. Column Constraint (Inline)

Bentuk paling sederhana dari check constraint adalah menuliskannya langsung setelah tipe data kolom. Pola ini sering disebut sebagai _inline constraint_ karena didefinisikan tepat di tempat kolom dideklarasikan.

```sql
-- Syntax paling sederhana
CREATE TABLE example (
    price NUMERIC CHECK (price > 0)
);
```

Pada contoh ini, kolom `price` hanya diperbolehkan menyimpan nilai yang lebih besar dari nol. Aturan ini langsung melekat pada kolom tersebut.

Untuk melihat bagaimana constraint ini bekerja, kita bisa mencoba memasukkan data yang valid.

```sql
INSERT INTO example VALUES (10);  -- ✅ Berhasil
```

Nilai `10` memenuhi kondisi `price > 0`, sehingga PostgreSQL menerima data tersebut tanpa masalah.

Sebaliknya, ketika kita mencoba memasukkan nilai yang melanggar aturan:

```sql
INSERT INTO example VALUES (-1);   -- ❌ ERROR!
```

PostgreSQL akan menolak operasi ini dan menampilkan error seperti berikut:

```text
ERROR: new row violates check constraint "example_price_check"
```

Nama constraint `example_price_check` dihasilkan otomatis oleh PostgreSQL. Meskipun masih bisa dipahami, nama ini kurang informatif, terutama jika tabel memiliki banyak constraint.

#### 2. Named Constraint (dengan Nama Custom)

Untuk meningkatkan kejelasan dan kemudahan debugging, kita bisa memberikan nama constraint secara eksplisit. Ini sangat disarankan untuk skema database yang serius atau dikelola oleh tim.

```sql
CREATE TABLE example (
    price NUMERIC CONSTRAINT price_must_be_positive CHECK (price > 0)
);
```

Di sini, constraint diberi nama `price_must_be_positive`, yang secara langsung menjelaskan aturan yang diterapkan.

Jika kita kembali mencoba memasukkan data yang tidak valid:

```sql
INSERT INTO example VALUES (-1);
```

Error yang muncul akan jauh lebih informatif:

```text
ERROR: new row violates check constraint "price_must_be_positive"
```

Dari pesan error saja, kita sudah tahu persis aturan mana yang dilanggar, tanpa perlu menebak-nebak atau membuka definisi tabel.

#### Keuntungan Memberi Nama pada Constraint

Memberi nama constraint bukan hanya soal kerapian, tetapi juga soal efisiensi jangka panjang. Beberapa keuntungannya antara lain:

- Pesan error menjadi lebih jelas dan mudah dipahami
- Memudahkan komunikasi antar anggota tim saat membahas masalah data
- Log file database menjadi lebih informatif
- Maintenance dan refactoring skema database menjadi lebih sederhana

Pendekatan ini semakin terasa manfaatnya ketika sebuah tabel memiliki lebih dari satu constraint.

```sql
-- Contoh lengkap dengan multiple constraints
CREATE TABLE example (
    price NUMERIC CONSTRAINT price_must_be_positive CHECK (price > 0),
    abbreviation TEXT CONSTRAINT abbr_length_five CHECK (length(abbreviation) = 5)
);
```

Pada tabel di atas, ada dua aturan sekaligus: `price` harus bernilai positif, dan `abbreviation` harus memiliki panjang tepat lima karakter.

Ketika diuji dengan data yang benar:

```sql
INSERT INTO example VALUES (10, 'ABCDE');  -- ✅ OK
```

Semua constraint terpenuhi, sehingga data berhasil disimpan.

Namun jika salah satu aturan dilanggar, PostgreSQL akan menolak data dan menyebutkan constraint yang relevan.

```sql
INSERT INTO example VALUES (10, 'ABC');    -- ❌ ERROR: abbr_length_five
INSERT INTO example VALUES (-5, 'ABCDE');  -- ❌ ERROR: price_must_be_positive
```

Dari pesan error tersebut, kita bisa langsung mengetahui kolom mana yang bermasalah dan aturan apa yang dilanggar. Inilah alasan mengapa named constraint sangat direkomendasikan dalam desain database yang rapi, jelas, dan mudah dirawat.

### c. Column Constraint vs Table Constraint

Dalam PostgreSQL, check constraint bisa dideklarasikan di dua level yang berbeda: langsung pada kolom (column constraint) atau pada level tabel (table constraint). Keduanya sama-sama berfungsi untuk memvalidasi data, tetapi memiliki peran dan batasan yang berbeda. Memahami perbedaannya penting agar desain constraint tetap jelas, aman, dan mudah dirawat.

#### Column Constraint

Column constraint dideklarasikan langsung setelah definisi sebuah kolom. Karena posisinya melekat pada kolom tertentu, jenis constraint ini hanya boleh mereferensikan kolom tersebut.

```sql
CREATE TABLE example (
    price NUMERIC CHECK (price > 0),                     -- Column constraint
    abbreviation TEXT CHECK (length(abbreviation) = 5)   -- Column constraint
);
```

Pada contoh di atas, setiap kolom memiliki aturan validasinya masing-masing. `price` harus bernilai positif, dan `abbreviation` harus memiliki panjang tepat lima karakter. Semua aturan ini berdiri sendiri dan tidak saling bergantung.

Karakteristik utama dari column constraint antara lain:

- Ringkas dan langsung terlihat di definisi kolom
- Mudah dibaca untuk aturan sederhana yang hanya melibatkan satu kolom
- Tidak bisa digunakan untuk membandingkan atau mengaitkan dengan kolom lain

Dengan kata lain, column constraint sangat cocok untuk validasi dasar seperti range angka, panjang string, atau format sederhana yang hanya berkaitan dengan satu kolom.

#### Table Constraint

Table constraint dideklarasikan setelah semua kolom selesai didefinisikan. Karena berada di level tabel, constraint ini bisa mereferensikan satu atau lebih kolom sekaligus.

```sql
CREATE TABLE example (
    price NUMERIC,
    abbreviation TEXT,
    -- Table constraints di sini
    CHECK (price > 0),
    CHECK (length(abbreviation) = 5)
);
```

Secara fungsi, contoh di atas setara dengan column constraint sebelumnya. Perbedaannya hanya pada lokasi penulisan. Namun kekuatan table constraint baru terasa ketika aturan validasi melibatkan lebih dari satu kolom.

#### Kapan Harus Menggunakan Table Constraint

Table constraint wajib digunakan ketika sebuah aturan membutuhkan perbandingan atau hubungan antar kolom.

```sql
CREATE TABLE example (
    price NUMERIC,
    discount_price NUMERIC,
    abbreviation TEXT,

    -- Column atau table constraints untuk validasi masing-masing kolom
    CHECK (price > 0),
    CHECK (discount_price > 0),
    CHECK (length(abbreviation) = 5),

    -- Table constraint karena melibatkan dua kolom
    CHECK (price > discount_price)
);
```

Constraint `CHECK (price > discount_price)` tidak mungkin ditulis sebagai column constraint, karena aturannya bergantung pada dua kolom sekaligus. PostgreSQL memang mengizinkan referensi lintas kolom hanya di level tabel.

#### Best Practice dalam Memilih Level Constraint

Kesalahan umum yang sering terjadi adalah mencoba menulis constraint multi-kolom di level kolom.

```sql
-- ❌ BURUK: Multi-column constraint di column level
CREATE TABLE products (
    price NUMERIC CHECK (price > discount_price),  -- SALAH: discount_price belum ada
    discount_price NUMERIC
);
```

Selain tidak valid secara sintaks, pendekatan ini juga membingungkan secara desain karena menyembunyikan relasi antar kolom di definisi satu kolom saja.

Pendekatan yang benar dan direkomendasikan adalah memindahkan aturan tersebut ke table constraint.

```sql
-- ✅ BAIK: Multi-column constraint di table level
CREATE TABLE products (
    price NUMERIC,
    discount_price NUMERIC,
    CHECK (price > discount_price)
);
```

Dengan cara ini, aturan bisnis yang melibatkan lebih dari satu kolom menjadi lebih jelas, eksplisit, dan mudah dipahami. Secara umum, gunakan column constraint untuk validasi sederhana per kolom, dan table constraint untuk aturan yang melibatkan relasi antar kolom.

### d. Batasan Check Constraints

Meskipun check constraint sangat berguna untuk menjaga validitas data, ada batasan penting yang perlu dipahami. Check constraint dirancang untuk melakukan validasi sederhana dan deterministik pada satu baris data saja. Begitu aturan validasi mulai bergantung pada data di luar baris tersebut, check constraint tidak lagi menjadi alat yang tepat.

#### 1. Tidak Bisa Mereferensikan Tabel Lain

Check constraint tidak diperbolehkan menggunakan subquery, termasuk query ke tabel lain. Artinya, kita tidak bisa membuat aturan yang bergantung pada data dari tabel berbeda.

```sql
-- ❌ TIDAK BISA: Referensi tabel lain
CREATE TABLE orders (
    product_id INTEGER,
    quantity INTEGER,
    CHECK (quantity <= (SELECT stock FROM products WHERE id = product_id))
);
```

Saat mencoba menjalankan definisi tabel di atas, PostgreSQL akan langsung menolak dengan error:

```text
ERROR: cannot use subquery in check constraint
```

Masalahnya bukan pada logika bisnisnya, melainkan pada keterbatasan check constraint itu sendiri. Check constraint harus selalu bisa dievaluasi secara lokal, tanpa bergantung pada kondisi eksternal yang bisa berubah kapan saja.

Jika kebutuhan bisnis memang mengharuskan validasi lintas tabel, pendekatan yang benar adalah menggunakan mekanisme lain seperti trigger atau kombinasi foreign key dan constraint tambahan.

#### 2. Tidak Bisa Mereferensikan Baris Lain dalam Tabel yang Sama

Selain tidak bisa mengakses tabel lain, check constraint juga tidak boleh membandingkan data dengan baris lain dalam tabel yang sama. Subquery ke tabel itu sendiri tetap dianggap subquery dan tidak diizinkan.

```sql
-- ❌ TIDAK BISA: Membandingkan dengan row lain
CREATE TABLE inventory (
    item_id INTEGER,
    quantity INTEGER,
    CHECK (quantity < (SELECT MAX(quantity) FROM inventory))
);
```

Definisi ini juga akan menghasilkan error yang sama:

```text
ERROR: cannot use subquery in check constraint
```

Alasannya konsisten: check constraint hanya boleh mengevaluasi nilai dari baris yang sedang di-_insert_ atau di-_update_. Ia tidak memiliki konteks global tentang isi tabel secara keseluruhan.

Aturan seperti “jumlah ini harus lebih kecil dari stok terbesar yang pernah ada” membutuhkan pemahaman lintas baris, sehingga harus ditangani oleh trigger atau logika aplikasi.

#### 3. Hanya Mengevaluasi Baris yang Sedang Diproses

Batasan ini sekaligus menjelaskan apa yang justru bisa dilakukan oleh check constraint. Selama semua nilai yang dibutuhkan berada dalam baris yang sama, constraint akan bekerja dengan baik.

```sql
-- ✅ BISA: Akses nilai dalam row yang sama
CREATE TABLE orders (
    price NUMERIC,
    tax NUMERIC,
    total NUMERIC,
    CHECK (total = price + tax)
);
```

Pada contoh ini, `price`, `tax`, dan `total` semuanya berada dalam satu baris yang sama. Ketika data dimasukkan atau diperbarui, PostgreSQL akan mengevaluasi ekspresi `total = price + tax` menggunakan nilai dari baris tersebut saja.

Jika hasilnya benar, data diterima. Jika tidak, operasi akan ditolak. Inilah pola penggunaan check constraint yang ideal: aturan sederhana, deterministik, dan sepenuhnya bergantung pada nilai dalam satu baris data.

### e. Memodifikasi Check Constraints

Salah satu hal penting yang perlu dipahami saat bekerja dengan check constraint adalah bahwa logika constraint tidak bisa diubah secara langsung. PostgreSQL tidak menyediakan mekanisme untuk “mengedit” ekspresi di dalam constraint. Jika aturan validasi perlu diubah, pendekatannya selalu sama: hapus constraint lama, lalu buat constraint baru dengan aturan yang diperbarui.

#### Tidak Bisa Menggunakan ALTER untuk Mengubah Logika Constraint

Sering kali muncul asumsi bahwa constraint bisa diubah seperti kolom, misalnya dengan perintah `ALTER CONSTRAINT`. Namun untuk check constraint, hal ini tidak didukung.

```sql
-- ❌ TIDAK BISA: Alter constraint logic
ALTER TABLE products ALTER CONSTRAINT price_positive CHECK (price > 10);
```

Perintah di atas akan langsung menghasilkan error sintaks, karena PostgreSQL memang tidak mengizinkan perubahan isi check constraint secara langsung. Constraint diperlakukan sebagai satu kesatuan utuh: jika isinya berubah, constraint tersebut harus diganti.

#### Pendekatan yang Benar: Drop dan Recreate Constraint

Untuk memperbarui aturan, constraint lama harus dihapus terlebih dahulu, lalu constraint baru ditambahkan dengan logika yang baru.

Cara paling sederhana adalah menggunakan dua statement terpisah.

```sql
-- Cara 1: Dua statement terpisah
ALTER TABLE products DROP CONSTRAINT price_positive;
ALTER TABLE products ADD CONSTRAINT price_positive CHECK (price > 10);
```

Pendekatan ini secara fungsional benar, tetapi memiliki risiko tersembunyi. Di antara dua perintah tersebut, terdapat jeda waktu di mana tabel tidak memiliki constraint sama sekali.

#### Risiko Window Tanpa Constraint

Jika dua statement dijalankan secara terpisah, akan ada momen singkat di mana constraint sudah dihapus tetapi belum ditambahkan kembali.

```sql
ALTER TABLE products DROP CONSTRAINT price_positive;
-- ⚠️ Pada titik ini, tabel tidak memiliki constraint
-- User bisa saja menginsert data invalid
ALTER TABLE products ADD CONSTRAINT price_positive CHECK (price > 10);
```

Dalam sistem dengan banyak koneksi atau aplikasi yang berjalan paralel, jeda singkat ini sudah cukup untuk memungkinkan data yang melanggar aturan masuk ke database.

#### Solusi yang Lebih Aman: Single Statement (Atomic)

Untuk menghindari masalah tersebut, PostgreSQL memungkinkan kita menggabungkan operasi drop dan add dalam satu perintah `ALTER TABLE`.

```sql
ALTER TABLE products
    DROP CONSTRAINT price_positive,
    ADD CONSTRAINT price_positive CHECK (price > 10);
```

Dalam bentuk ini, perubahan dilakukan secara atomik. Artinya, PostgreSQL memperlakukan operasi tersebut sebagai satu kesatuan. Tidak pernah ada kondisi di mana constraint benar-benar “hilang” di tengah proses.

Pendekatan single statement ini sangat direkomendasikan, terutama pada database produksi, karena memastikan integritas data tetap terjaga sepanjang proses perubahan aturan.

### f. Business Logic vs Data Integrity

Salah satu diskusi klasik dalam desain sistem adalah menentukan batas antara tanggung jawab database dan tanggung jawab aplikasi. Pertanyaannya sederhana, tetapi jawabannya tidak selalu hitam-putih: aturan mana yang sebaiknya ditegakkan oleh database, dan aturan mana yang lebih tepat diletakkan di application layer?

Kuncinya ada pada pembedaan antara **data integrity** dan **business logic**.

#### Data Integrity (Cocok Ditegakkan di Database)

Data integrity berkaitan dengan konsistensi dan validitas dasar data. Aturan-aturan ini biasanya bersifat struktural, jarang berubah, dan harus selalu benar apa pun aplikasi yang mengakses database.

```sql
CREATE TABLE products (
    price NUMERIC CHECK (price > 0),              -- Harga harus positif
    stock INTEGER CHECK (stock >= 0),              -- Stok tidak boleh negatif
    sku TEXT CHECK (length(sku) = 8),              -- SKU harus 8 karakter
    discount_price NUMERIC,
    CHECK (price >= discount_price)                -- Harga >= harga diskon
);
```

Constraint di atas bukan tentang “bagaimana bisnis berjalan”, tetapi tentang memastikan data masuk akal. Harga tidak boleh negatif, stok tidak boleh minus, dan harga diskon tidak boleh melebihi harga normal. Aturan seperti ini hampir pasti berlaku untuk semua aplikasi yang menggunakan tabel tersebut, sekarang maupun di masa depan.

Karena sifatnya fundamental, database adalah tempat yang tepat untuk menegakkannya.

#### Business Logic (Perlu Dipertimbangkan di Application Layer)

Business logic biasanya mencerminkan kebijakan atau strategi bisnis. Aturan jenis ini sering kali lebih kompleks, bisa berubah seiring waktu, dan kadang bergantung pada konteks eksternal.

```sql
CREATE TABLE orders (
    subtotal NUMERIC,
    discount NUMERIC,
    CHECK (
        CASE
            WHEN subtotal > 1000 THEN discount <= subtotal * 0.2
            WHEN subtotal > 500 THEN discount <= subtotal * 0.1
            ELSE discount <= subtotal * 0.05
        END
    )
);
```

Secara teknis, constraint di atas valid dan akan bekerja. Namun, kompleksitasnya cukup tinggi dan sangat mungkin berubah: batas subtotal bisa diubah, persentase diskon bisa disesuaikan, atau logikanya menjadi lebih rumit.

Jika aturan seperti ini ditanam langsung di database, setiap perubahan kebijakan bisnis akan memaksa kita melakukan `ALTER TABLE` dan memodifikasi constraint. Dalam banyak kasus, hal ini lebih fleksibel jika ditangani di application layer.

#### Rekomendasi Umum

Sebagai panduan praktis, berikut cara membagi tanggung jawab antara database dan aplikasi:

| Kriteria                           | Database (CHECK) | Application |
| ---------------------------------- | ---------------- | ----------- |
| Konsistensi data secara struktural | Cocok            |             |
| Rule yang jarang berubah           | Cocok            |             |
| Database diakses banyak aplikasi   | Cocok            |             |
| Perhitungan bisnis yang kompleks   |                  | Cocok       |
| Rule yang sering berubah           |                  | Cocok       |
| Membutuhkan data eksternal         |                  | Cocok       |

Pendekatan ini membantu menjaga database tetap kuat dan konsisten, tanpa menjadikannya terlalu kaku atau sulit dikembangkan.

#### Contoh yang Jelas Masuk Data Integrity

Beberapa aturan sering disalahartikan sebagai business logic, padahal sebenarnya murni soal validitas data.

```sql
CREATE TABLE users (
    age INTEGER CHECK (age >= 0 AND age <= 150),
    email TEXT CHECK (email LIKE '%@%')
);
```

Aturan usia masuk akal secara biologis, dan format email minimal harus mengandung karakter `@`. Ini bukan kebijakan bisnis, melainkan perlindungan terhadap data yang tidak valid.

#### Contoh yang Lebih Tepat Dipertimbangkan di Aplikasi

Sebaliknya, ada aturan yang terlihat cocok sebagai constraint, tetapi memiliki potensi perubahan tinggi.

```sql
CREATE TABLE memberships (
    tier TEXT CHECK (tier IN ('bronze', 'silver', 'gold', 'platinum')),
    discount_percent NUMERIC CHECK (
        (tier = 'platinum' AND discount_percent <= 20) OR
        (tier = 'gold' AND discount_percent <= 15) OR
        (tier = 'silver' AND discount_percent <= 10) OR
        (tier = 'bronze' AND discount_percent <= 5)
    )
);
```

Selama daftar tier dan persentase diskon stabil, pendekatan ini masih masuk akal. Namun jika bisnis sering menambah tier baru atau mengubah besaran diskon, constraint ini akan sering dimodifikasi.

Dalam situasi seperti itu, menempatkan aturan di application layer biasanya lebih fleksibel dan lebih mudah disesuaikan dengan kebutuhan bisnis yang terus berkembang.

## 3. Hubungan Antar Konsep

### Flow Pemilihan Constraint Type:

```
Apakah constraint melibatkan multiple kolom?
├─ Ya → HARUS Table Constraint
│   └─ Contoh: CHECK (price > discount_price)
└─ Tidak → Boleh Column atau Table Constraint (preferensi style)
    ├─ Column Constraint (inline, ringkas)
    │   └─ Contoh: price NUMERIC CHECK (price > 0)
    └─ Table Constraint (terpisah, konsisten dengan multi-column)
        └─ Contoh: CHECK (price > 0)
```

### Evolusi dari Video Sebelumnya:

```
Video: Character Types
    Problem: CHAR(5) tidak enforce panjang tepat 5
    ↓
Video: Check Constraints
    Solution: TEXT + CHECK (length(col) = 5)

Video: Numeric Types
    Problem: Tidak ada UNSIGNED INTEGER
    ↓
Video: Check Constraints
    Solution: INTEGER + CHECK (col >= 0)
```

### Hirarki Data Protection:

```
Level 1: Tipe Data
    ├─ INTEGER (bukan TEXT)
    └─ NUMERIC (bukan VARCHAR)

Level 2: Check Constraints
    ├─ CHECK (price > 0)
    └─ CHECK (length(sku) = 8)

Level 3: Foreign Keys (belum dibahas)
    └─ REFERENCES other_table(id)

Level 4: Triggers (complex logic)
    └─ BEFORE INSERT/UPDATE
```

## 4. Kesimpulan

Check constraints adalah tool yang powerful untuk menerapkan data integrity di level database. Mereka memungkinkan kita membuat aturan validasi yang lebih spesifik dari yang bisa dilakukan hanya dengan tipe data saja.

**Key Points:**

1. **Syntax**: Bisa sebagai column constraint (inline) atau table constraint (terpisah)
2. **Naming**: Selalu beri nama untuk memudahkan debugging
3. **Multi-column rules**: HARUS menggunakan table constraint
4. **Limitations**: Tidak bisa referensi tabel lain atau row lain
5. **Modification**: Harus DROP dan RECREATE, preferably dalam single statement

**Best Practices:**

- Gunakan check constraints untuk **data integrity** (bukan complex business logic)
- Beri nama yang descriptive untuk semua constraints
- Pindahkan multi-column constraints ke table level
- Kombinasikan dengan NOT NULL jika tidak mau terima NULL (check constraint masih bisa menerima NULL)
- Pertimbangkan performa untuk checks yang complex

**Philosophy:**
Ada perdebatan tentang seberapa banyak logic yang masuk database. Untuk **data integrity** (seperti "harga harus positif", "email harus valid", "panjang harus tepat 5"), check constraints adalah tempat yang tepat. Untuk **business logic** yang complex dan sering berubah, pertimbangkan aplikasi layer.

**Key takeaway:** Check constraints mengisi gap antara tipe data dasar dan kebutuhan validasi bisnis. Mereka adalah layer pertahanan pertama untuk data quality, terutama ketika multiple aplikasi atau tools mengakses database yang sama.
