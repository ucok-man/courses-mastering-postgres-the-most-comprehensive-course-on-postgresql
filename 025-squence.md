# 📝 Sequences di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas cara membuat dan memanipulasi **sequences** secara manual di PostgreSQL (tidak terikat dengan serial column). Sequences adalah object database yang menghasilkan angka berurutan dan dapat digunakan independen untuk berbagai keperluan. Video ini menjelaskan cara membuat sequence, fungsi-fungsi untuk memanipulasinya (`nextval`, `currval`, `setval`), dan perilaku sequence dalam konteks session yang berbeda.

## 2. Konsep Utama

### a. Membuat Sequence Manual

Berbeda dengan `SERIAL` yang otomatis membuat sequence di belakang layar, kita juga bisa membuat sequence secara manual. Pendekatan ini biasanya dipakai ketika kita butuh kontrol yang lebih spesifik — misalnya ingin menentukan angka awal tertentu, increment khusus, atau memberi batas maksimum.

Dengan membuat sequence secara eksplisit, kita punya fleksibilitas penuh terhadap perilakunya.

#### Syntax Dasar

Secara umum, bentuk paling sederhana dari sequence adalah:

```sql
CREATE SEQUENCE sequence_name AS data_type;
```

Di sini kita hanya menentukan nama sequence dan tipe datanya. Namun dalam praktik nyata, kita hampir selalu menambahkan beberapa opsi tambahan agar sequence bekerja sesuai kebutuhan.

#### Contoh dengan Opsi Lengkap

Berikut contoh sequence dengan konfigurasi lengkap:

```sql
CREATE SEQUENCE my_sequence AS BIGINT
    INCREMENT BY 1
    START WITH 1
    MINVALUE 1
    MAXVALUE 9223372036854775807
    CACHE 1;
```

Mari kita pahami satu per satu:

- `AS BIGINT` → menentukan tipe data sequence. Karena `BIGINT`, maka batas maksimumnya sangat besar.
- `INCREMENT BY 1` → setiap kali dipanggil, nilai akan bertambah 1.
- `START WITH 1` → nilai pertama yang dihasilkan adalah 1.
- `MINVALUE 1` → nilai minimum yang diizinkan.
- `MAXVALUE 9223372036854775807` → batas maksimum sesuai kapasitas BIGINT.
- `CACHE 1` → hanya menyimpan 1 nilai di memory (tidak banyak caching).

Konfigurasi seperti ini sebenarnya mirip dengan default behavior, tetapi dituliskan secara eksplisit agar lebih jelas dan terkontrol.

#### Penjelasan Parameter Penting

Berikut parameter yang paling sering digunakan beserta penjelasan praktisnya:

- `INCREMENT BY`
  Default: `1`
  Menentukan berapa kenaikan (atau penurunan) setiap kali sequence dipanggil.
  Misalnya:

  ```sql
  INCREMENT BY 10
  ```

  Maka nilai yang dihasilkan akan 10, 20, 30, dan seterusnya.

- `START WITH`
  Default: `1`
  Menentukan angka awal.
  Cocok untuk kebutuhan seperti nomor invoice yang ingin dimulai dari angka tertentu, misalnya 1000.

- `MINVALUE`
  Default: `1` (untuk ascending sequence)
  Menentukan batas minimum. Berguna jika sequence bisa turun (increment negatif).

- `MAXVALUE`
  Default: maksimum sesuai tipe data
  Digunakan untuk membatasi angka agar tidak melewati batas tertentu.

- `CACHE`
  Default: `1`
  Menentukan berapa banyak nilai yang disimpan di memory untuk meningkatkan performa.
  Semakin besar cache, semakin cepat performa, tetapi ada risiko gap jika database crash.

#### Contoh Praktis Penggunaan

Sekarang kita lihat beberapa contoh nyata agar lebih mudah dipahami.

##### 1. Sequence untuk Nomor Order Dimulai dari 1000

```sql
CREATE SEQUENCE order_number_seq AS INTEGER
    START WITH 1000
    INCREMENT BY 1;
```

Artinya:

- Nilai pertama yang keluar adalah 1000.
- Berikutnya 1001, 1002, 1003, dan seterusnya.

Ini cocok untuk sistem e-commerce yang ingin nomor order terlihat profesional dan tidak dimulai dari angka kecil.

##### 2. Sequence dengan Kenaikan Kelipatan 10

```sql
CREATE SEQUENCE ticket_seq AS INTEGER
    START WITH 10
    INCREMENT BY 10;
```

Jika kita panggil berulang kali, hasilnya:

```
10, 20, 30, 40, ...
```

Ini bisa dipakai jika kita ingin membuat nomor antrian atau batch number dengan pola tertentu.

##### 3. Sequence dengan Batas Maksimum

```sql
CREATE SEQUENCE limited_seq AS INTEGER
    START WITH 1
    MAXVALUE 100;
```

Sequence ini:

- Dimulai dari 1.
- Akan terus naik sampai 100.
- Setelah mencapai 100, jika dipanggil lagi, database akan menghasilkan error karena melewati `MAXVALUE`.

Use case seperti ini cocok untuk sistem yang memang ingin membatasi jumlah maksimum tertentu, misalnya nomor kupon terbatas.

#### Kesimpulan Konseptual

Membuat sequence manual memberi kita kontrol penuh terhadap:

- Dari angka berapa mulai
- Seberapa besar kenaikannya
- Sampai angka berapa diperbolehkan
- Bagaimana performanya melalui cache

Berbeda dengan `SERIAL` yang praktis dan otomatis, manual sequence lebih fleksibel dan cocok untuk kebutuhan bisnis yang spesifik. Dalam sistem produksi yang kompleks, pendekatan ini sering dipakai agar pola numbering sesuai dengan aturan domain bisnis.

### b. START WITH: Import Data dari Sistem Lain

Parameter `START WITH` menjadi sangat penting ketika kita melakukan migrasi data atau import dari sistem lain (misalnya sistem legacy). Dalam situasi seperti ini, biasanya data lama sudah memiliki ID sendiri, dan kita harus memastikan sequence baru tidak menghasilkan ID yang bentrok.

Intinya sederhana: sequence harus dimulai dari angka setelah ID terbesar yang sudah ada.

Mari kita bahas skenarionya secara runtut.

#### Use Case: Import dari Sistem Legacy

Misalnya kita memiliki sistem lama yang sudah menyimpan data user dengan ID sampai 5432. Kita ingin memindahkan data tersebut ke database baru yang menggunakan sequence untuk auto increment.

Langkahnya biasanya seperti ini.

##### 1. Import Data Lama dengan ID Existing

Pertama, kita masukkan semua data lama apa adanya, termasuk nilai ID-nya.

```sql
INSERT INTO users (id, name) VALUES
    (1, 'Old User 1'),
    (2, 'Old User 2'),
    -- ...
    (5432, 'Old User 5432');
```

Di tahap ini:

- Kita tidak menggunakan sequence.
- Kita mempertahankan ID asli dari sistem lama.
- Tujuannya agar tidak mengubah referensi data (misalnya relasi antar tabel).

Setelah langkah ini selesai, tabel `users` sudah berisi ID dari 1 sampai 5432.

##### 2. Buat Sequence Dimulai Setelah Data Terakhir

Sekarang kita perlu membuat sequence baru. Kuncinya adalah: sequence harus dimulai dari 5433, yaitu satu angka lebih tinggi dari ID terbesar yang sudah ada.

```sql
CREATE SEQUENCE users_id_seq AS BIGINT
    START WITH 5433;
```

Kenapa 5433?

Karena jika kita mulai dari angka yang lebih kecil atau sama dengan 5432, maka saat insert baru, akan terjadi bentrok primary key.

Secara konsep:

- ID lama: 1 → 5432
- ID baru harus mulai dari: 5433

Inilah fungsi utama `START WITH` dalam konteks migrasi data.

##### 3. Jadikan Sequence sebagai Default Kolom

Sequence sudah dibuat, tetapi belum terhubung dengan kolom `id`. Agar setiap insert baru otomatis menggunakan sequence, kita harus mengatur default value pada kolom tersebut.

```sql
ALTER TABLE users
ALTER COLUMN id SET DEFAULT nextval('users_id_seq');
```

Perintah ini berarti:

- Jika kita tidak mengisi kolom `id` saat insert,
- Maka database akan memanggil `nextval('users_id_seq')`,
- Dan menghasilkan angka berikutnya dari sequence.

Sekarang kolom `id` sudah bersifat auto increment, tetapi dengan titik awal yang aman.

##### 4. Insert Data Baru

Sekarang kita coba insert data baru tanpa menyebutkan ID.

```sql
INSERT INTO users (name) VALUES ('New User');
```

Apa yang terjadi?

- Database akan memanggil `nextval('users_id_seq')`.
- Karena `START WITH 5433`, maka ID pertama yang keluar adalah 5433.
- Insert berikutnya akan menjadi 5434, 5435, dan seterusnya.

Hasilnya:

```
ID: 5433
```

Dan tidak ada konflik dengan data lama.

#### Mengapa Ini Penting?

Pengaturan `START WITH` dalam skenario migrasi sangat krusial karena:

- Menghindari duplicate key error
  Jika sequence dimulai dari angka yang sudah ada, insert baru akan gagal karena primary key sudah terpakai.

- Membuat transisi dari sistem lama menjadi seamless
  Kita tidak perlu mengubah atau merombak ID lama.

- Tidak perlu renumber data existing
  Mengubah ID lama sangat berisiko, terutama jika ada relasi ke tabel lain.

Secara praktis, pola yang aman saat migrasi adalah:

1. Import data lama.
2. Cek nilai maksimum ID (misalnya dengan `SELECT MAX(id) FROM users;`).
3. Buat sequence dengan `START WITH max_id + 1`.
4. Set sebagai default.

Dengan cara ini, sistem baru bisa langsung berjalan tanpa konflik, dan integritas data tetap terjaga.

### c. CACHE Parameter

Salah satu parameter penting dalam sequence yang sering diabaikan adalah `CACHE`. Parameter ini berkaitan langsung dengan performa dan juga konsekuensi berupa kemungkinan adanya “gap” pada angka sequence.

Memahami `CACHE` itu penting, terutama jika kita bekerja di sistem dengan traffic tinggi atau sistem yang sensitif terhadap urutan nomor.

#### Apa itu CACHE

Secara sederhana, `CACHE` menentukan berapa banyak nilai sequence yang disiapkan (pre-allocate) terlebih dahulu di memory.

Beberapa poin penting:

- Menentukan jumlah nilai yang di-pre-allocate di memory.
- Default-nya adalah `1` (artinya hampir tidak ada caching).
- Ada trade-off antara performa dan potensi adanya gap angka.

Ketika `CACHE` bernilai kecil, database akan lebih sering menulis ke disk. Ketika `CACHE` besar, database lebih jarang melakukan I/O ke disk, sehingga performa meningkat.

#### Cara Kerja CACHE

Misalnya kita membuat sequence seperti ini:

```sql
CREATE SEQUENCE cached_seq AS INTEGER
    CACHE 20;
```

Apa yang terjadi di belakang layar?

1. PostgreSQL akan langsung menyiapkan nilai 1–20 di memory.
2. Setiap kali `nextval()` dipanggil, database cukup mengambil dari memory (tidak perlu ke disk).
3. Setelah nilai ke-20 habis, PostgreSQL akan menyiapkan lagi 21–40.
4. Proses ini terus berulang.

Karena sebagian besar operasi dilakukan di memory, maka:

- Akses menjadi lebih cepat.
- Beban I/O ke disk berkurang.
- Cocok untuk sistem dengan banyak operasi insert.

Jadi, semakin besar nilai `CACHE`, semakin jarang database perlu sinkronisasi ke disk.

#### Trade-off: Performance vs Gaps

Sekarang kita masuk ke bagian penting: risikonya.

Misalnya kita menggunakan:

```sql
CACHE 20
```

Skenario:

- PostgreSQL sudah mengalokasikan nilai 1–20 di memory.
- Baru digunakan sampai nilai 5.
- Tiba-tiba server crash.

Apa yang terjadi?

Nilai 6–20 belum sempat digunakan, tetapi sudah dianggap “dipakai” oleh sistem karena sudah dialokasikan. Setelah restart:

- Sequence tidak akan kembali ke 6.
- Nilai berikutnya langsung 21.

Artinya, akan muncul gap: 6–20 tidak pernah dipakai.

Sekarang bandingkan dengan default:

```sql
CACHE 1
```

Dalam konfigurasi ini:

- Setiap kali `nextval()` dipanggil, PostgreSQL akan melakukan disk write.
- Lebih lambat karena ada I/O setiap kali increment.
- Tetapi lebih aman terhadap gap akibat crash.

Perlu dipahami: bahkan dengan `CACHE 1`, gap tetap bisa terjadi dalam kondisi tertentu (misalnya transaksi rollback). Sequence memang tidak menjamin angka tanpa celah. Namun, `CACHE` besar memperbesar potensi gap saat crash.

#### Rekomendasi Penggunaan

Secara umum:

- Default (`CACHE 1`)
  Cocok untuk sebagian besar aplikasi. Aman dan sederhana. Biasanya sudah cukup untuk sistem dengan traffic normal.

- CACHE lebih tinggi
  Cocok untuk high-throughput systems, misalnya:
  - Sistem logging besar
  - Event processing
  - Aplikasi dengan ribuan insert per detik

  Dengan catatan: sistem tersebut tidak mempermasalahkan adanya gap pada nomor.

#### Perspektif Desain Sistem

Penting untuk memahami bahwa sequence bukanlah alat untuk membuat nomor yang harus benar-benar berurutan tanpa celah (misalnya nomor faktur legal di beberapa negara).

Sequence dirancang untuk:

- Menghasilkan nilai unik.
- Cepat dan scalable.
- Aman dari race condition.

Jika bisnis benar-benar membutuhkan nomor tanpa gap, pendekatannya berbeda dan biasanya jauh lebih kompleks.

Jadi saat memilih nilai `CACHE`, tanyakan:

- Apakah performa sangat krusial?
- Apakah gap nomor menjadi masalah bisnis?

Jawaban atas dua pertanyaan itu akan menentukan konfigurasi terbaik untuk sistem Anda.

### d. nextval(): Mendapatkan Nilai Berikutnya

Fungsi `nextval()` adalah komponen paling penting ketika kita bekerja dengan **sequence** di SQL. Fungsinya sederhana tapi krusial: mengambil nilai berikutnya dari sebuah sequence sekaligus menaikkan nilainya.

#### Konsep Dasar

Ketika kamu memanggil `nextval()`, database akan:

1. Mengambil nilai saat ini dari sequence
2. Mengembalikan nilai tersebut ke kamu
3. Langsung menaikkan nilai sequence untuk pemanggilan berikutnya

Semua itu terjadi dalam satu langkah yang tidak terpisahkan.

#### Syntax

```sql
SELECT nextval('sequence_name');
```

Di sini, `'sequence_name'` adalah nama sequence yang sudah kamu buat sebelumnya.

#### Contoh Penggunaan

```sql
CREATE SEQUENCE demo_seq AS INTEGER;

-- Panggilan berturut-turut
SELECT nextval('demo_seq');  -- Output: 1
SELECT nextval('demo_seq');  -- Output: 2
SELECT nextval('demo_seq');  -- Output: 3
SELECT nextval('demo_seq');  -- Output: 4
```

Penjelasannya:

- Saat pertama kali dipanggil, `nextval('demo_seq')` menghasilkan `1`
- Panggilan kedua menghasilkan `2`
- Dan seterusnya, selalu naik satu per pemanggilan

Sequence ini berfungsi seperti counter otomatis yang terus bertambah.

#### Karakteristik Penting

Beberapa hal yang wajib kamu pahami tentang `nextval()`:

- **Mengambil sekaligus menaikkan nilai (atomic operation)**
  Operasi ini tidak bisa “setengah jalan”. Begitu dipanggil, nilai langsung diambil dan sequence langsung bertambah. Tidak ada risiko dua proses mendapatkan angka yang sama.

- **Tidak bisa di-rollback**
  Ini sering jadi jebakan untuk pemula.
  Misalnya:
  - Kamu memanggil `nextval()` dalam sebuah transaksi
  - Tapi transaksi tersebut gagal (rollback)

  Nilai sequence **tetap dianggap sudah terpakai**. Jadi akan ada “loncatan angka” (gap), dan itu normal.

- **Aman untuk concurrent (thread-safe)**
  Kalau ada banyak user atau proses yang memanggil `nextval()` secara bersamaan, database tetap menjamin:
  - Tidak ada duplikasi nilai
  - Setiap pemanggilan tetap unik

#### Dalam Konteks INSERT

Salah satu penggunaan paling umum `nextval()` adalah untuk membuat ID otomatis.

```sql
CREATE TABLE orders (
    id INTEGER DEFAULT nextval('order_seq'),
    order_date DATE
);

INSERT INTO orders (order_date) VALUES (CURRENT_DATE);
```

Penjelasannya:

- Kolom `id` memiliki default value dari `nextval('order_seq')`
- Saat kamu melakukan `INSERT` tanpa menyebutkan `id`, database otomatis:
  - Memanggil `nextval('order_seq')`
  - Mengisi kolom `id` dengan nilai tersebut

Jadi kamu tidak perlu lagi mengatur ID secara manual.

#### Gambaran Praktis

Bayangkan sequence seperti mesin nomor antrian:

- Setiap kali kamu tekan tombol (`nextval()`), keluar nomor baru
- Nomor itu tidak bisa “dikembalikan” meskipun kamu batal antre
- Banyak orang bisa ambil nomor bersamaan tanpa bentrok

Dengan memahami ini, kamu bisa menggunakan sequence dengan lebih percaya diri, terutama untuk kebutuhan seperti auto-increment ID di database.

### e. currval(): Mendapatkan Nilai Terakhir dalam Session

Fungsi `currval()` digunakan untuk mengambil **nilai terakhir yang sebelumnya dihasilkan oleh `nextval()` dalam session yang sama**. Ini penting: `currval()` tidak membaca nilai global terbaru dari sequence, melainkan hanya “mengingat” apa yang terakhir kamu ambil di session tersebut.

#### Konsep Dasar

Cara kerja `currval()` bisa dibayangkan seperti “catatan lokal” di dalam session:

- Saat kamu memanggil `nextval()`, nilainya disimpan di session
- `currval()` hanya mengambil nilai terakhir dari catatan tersebut
- Jika belum pernah memanggil `nextval()`, maka `currval()` tidak punya data

#### Contoh Dasar

```sql
-- Session A
SELECT nextval('demo_seq');  -- Output: 10
SELECT currval('demo_seq');  -- Output: 10
SELECT currval('demo_seq');  -- Output: 10 (tetap sama)
```

Penjelasan:

- `nextval('demo_seq')` menghasilkan nilai `10`
- `currval('demo_seq')` akan selalu mengembalikan `10`, selama belum ada `nextval()` baru
- Tidak ada increment di sini — `currval()` hanya membaca, bukan mengubah

#### Perilaku Multi-Session

Hal paling penting dari `currval()` adalah sifatnya yang **session-specific**.

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
```

Penjelasan:

- Walaupun Session B sudah menaikkan sequence sampai 44
- Session A tetap “melihat” nilai terakhirnya sendiri, yaitu 28
- `currval()` tidak peduli dengan perubahan dari session lain

#### Visualisasi Konsep

```
Global Sequence State: 1 → 2 → 3 → ... → 44

Session A:                   Session B:
├─ nextval() → 28           ├─ nextval() → 29
├─ currval() → 28           ├─ nextval() → 30
├─ currval() → 28           ├─ nextval() → 31
└─ currval() → 28           └─ ... → 44
   (tetap 28!)
```

Dari sini terlihat jelas:

- Sequence global terus berjalan
- Tapi setiap session punya “view” sendiri terhadap nilai terakhirnya

#### Use Case Praktis

Salah satu penggunaan paling umum `currval()` adalah saat kamu ingin menggunakan ID yang sama untuk beberapa operasi dalam satu transaksi.

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

Penjelasan:

- Saat `INSERT` ke `orders`, sequence `order_seq` dipanggil (melalui default `nextval`)
- Nilai ID (misalnya 100) tersimpan di session
- `currval('order_seq')` digunakan untuk memastikan semua `order_items` terhubung ke order yang sama

Ini sangat membantu untuk menjaga **relasi antar tabel tetap konsisten** tanpa perlu menyimpan ID di aplikasi.

#### Error Jika Belum Ada nextval

```sql
-- Session baru, belum pernah nextval
SELECT currval('demo_seq');
-- ERROR: currval of sequence "demo_seq" is not yet defined in this session

-- Harus nextval dulu
SELECT nextval('demo_seq');  -- Output: 50
SELECT currval('demo_seq');  -- Output: 50 ✅
```

Penjelasan:

- `currval()` tidak bisa berdiri sendiri
- Harus ada pemanggilan `nextval()` sebelumnya di session yang sama
- Kalau tidak, database akan error karena tidak tahu nilai apa yang harus dikembalikan

#### Kapan Menggunakan currval

Gunakan `currval()` ketika:

- Kamu berada dalam **transaction yang sama** dan butuh nilai yang konsisten
- Ingin menghindari menyimpan ID di sisi aplikasi
- Membuat relasi antar tabel (parent-child) secara aman
- Memastikan semua operasi menggunakan ID yang sama tanpa risiko mismatch

Dengan memahami perbedaan antara `nextval()` dan `currval()`, kamu bisa mengelola sequence dengan lebih presisi — terutama dalam skenario concurrent dan transaksi kompleks.

### f. setval(): Mereset atau Mengatur Sequence

Fungsi `setval()` digunakan ketika kamu ingin **mengatur ulang atau memindahkan posisi sequence secara manual**. Berbeda dengan `nextval()` yang hanya mengambil nilai berikutnya, `setval()` memberi kamu kontrol langsung terhadap angka yang akan digunakan sequence ke depan.

#### Konsep Dasar

Saat kamu menggunakan `setval()`, kamu sebenarnya sedang berkata ke database:

> “Mulai sekarang, anggap nilai terakhir sequence adalah ini.”

Artinya, pemanggilan `nextval()` berikutnya akan melanjutkan dari nilai yang kamu set.

#### Syntax

```sql
SELECT setval('sequence_name', new_value);
```

- `sequence_name`: nama sequence
- `new_value`: nilai yang ingin kamu jadikan sebagai posisi terakhir sequence

#### Contoh Penggunaan

##### 1. Reset ke awal

```sql
-- Sequence saat ini di 45
SELECT setval('demo_seq', 1);

SELECT nextval('demo_seq');  -- Output: 2
```

Penjelasan:

- Kamu mengatur sequence ke `1`
- Tapi karena `setval()` menganggap itu adalah **nilai terakhir yang sudah dipakai**
- Maka `nextval()` berikutnya menghasilkan `2`, bukan `1`

##### 2. Set ke nilai tinggi

```sql
-- Jump ke nilai tinggi
SELECT setval('demo_seq', 10000);

SELECT nextval('demo_seq');  -- Output: 10001
```

Penjelasan:

- Sequence “meloncat” ke angka besar
- Cocok untuk skenario tertentu seperti migrasi atau penggabungan data

##### 3. Memperbaiki setelah manual insert

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

Penjelasan:

- Karena kamu memasukkan ID secara manual (`999`), sequence tidak tahu tentang itu
- Kalau tidak diperbaiki, `nextval()` bisa menghasilkan ID yang sudah ada → error
- `setval()` menyelaraskan sequence dengan data aktual

#### ⚠️ Bahaya setval()

`setval()` sangat powerful, tapi juga berbahaya jika digunakan sembarangan.

```sql
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
```

Penjelasan:

- Kamu menurunkan sequence ke nilai yang sudah dipakai
- `nextval()` akan menghasilkan ID yang sudah ada di tabel
- Akibatnya terjadi **duplicate key error**

Intinya: jangan pernah set sequence ke nilai yang lebih rendah dari data yang sudah ada, kecuali kamu benar-benar tahu konsekuensinya.

#### Use Case yang Valid

Beberapa situasi di mana `setval()` memang diperlukan:

1. **Setelah import data**
   Misalnya kamu import data dari file atau database lain yang sudah punya ID tertentu

2. **Maintenance database**
   Untuk memperbaiki sequence yang tidak sinkron dengan data

3. **Testing**
   Reset sequence agar mudah melakukan pengujian berulang

4. **Migration**
   Menyamakan nilai sequence dengan sistem lain saat proses migrasi

#### Best Practice

Sebelum menggunakan `setval()`, pastikan kamu tahu nilai maksimum yang sudah ada di tabel.

```sql
-- Cari max ID yang ada
SELECT MAX(id) FROM users;  -- Misalnya: 5432

-- Set sequence ke nilai tersebut
SELECT setval('users_id_seq', 5432);
```

Atau langsung dengan query dinamis:

```sql
SELECT setval('users_id_seq', (SELECT MAX(id) FROM users));
```

Penjelasan:

- Dengan cara ini, kamu memastikan sequence selalu “lanjut” dari ID terbesar yang sudah ada
- Menghindari konflik dan menjaga konsistensi data

Dengan memahami `setval()`, kamu bisa mengontrol sequence secara fleksibel. Tapi ingat, karena sifatnya manual, fungsi ini harus digunakan dengan hati-hati agar tidak merusak integritas data di database.

### g. Sequence dalam Multiple Sessions

Salah satu hal penting yang perlu dipahami adalah bahwa **sequence merupakan shared object**, artinya bisa digunakan oleh banyak session (atau koneksi) secara bersamaan.

Berbeda dengan variabel biasa yang hanya hidup di satu session, sequence bersifat global — semua session akan mengakses sumber yang sama.

#### Konsep Dasar

Ketika beberapa session memanggil `nextval()` secara bersamaan:

- Semua akan mengambil angka dari sequence yang sama
- Nilai akan tetap unik dan berurutan
- Tidak akan terjadi bentrok atau duplikasi

#### Demonstrasi

```sql id="k6i9pt"
-- Terminal/Session 1
SELECT nextval('demo_seq');  -- 24
SELECT nextval('demo_seq');  -- 25

-- Terminal/Session 2 (concurrent)
SELECT nextval('demo_seq');  -- 26
SELECT nextval('demo_seq');  -- 27

-- Kembali ke Session 1
SELECT nextval('demo_seq');  -- 28 (lanjut dari Session 2)
```

Penjelasan:

- Session 1 mengambil nilai 24 dan 25
- Session 2, walaupun berbeda koneksi, langsung melanjutkan ke 26 dan 27
- Ketika kembali ke Session 1, nilainya sudah di 28

Ini menunjukkan bahwa sequence benar-benar **global dan konsisten lintas session**.

#### Karakteristik Penting

Beberapa sifat utama sequence dalam lingkungan multi-session:

- **Thread-safe**
  Banyak proses bisa memanggil `nextval()` secara bersamaan tanpa risiko mendapatkan nilai yang sama.

- **Atomic**
  Setiap pemanggilan `nextval()` adalah satu operasi utuh:
  - Ambil nilai
  - Naikkan sequence
    Tidak bisa “terpotong” di tengah jalan.

- **Tanpa locking manual**
  Kamu tidak perlu menambahkan `LOCK` atau mekanisme sinkronisasi sendiri. Database sudah menanganinya secara internal.

- **Performa tinggi**
  Sequence didesain untuk sangat cepat, bahkan dalam kondisi banyak concurrent request.

#### Perbandingan dengan currval()

Walaupun `nextval()` bersifat global, `currval()` tetap bersifat lokal (per session). Ini sering jadi sumber kebingungan.

```sql id="c2h7sl"
-- Session A
SELECT nextval('demo_seq');  -- 10
-- (global sequence sekarang: 10)

-- Session B
SELECT nextval('demo_seq');  -- 11
SELECT nextval('demo_seq');  -- 12
-- (global sequence sekarang: 12)

-- Session A
SELECT currval('demo_seq');  -- Tetap 10!

SELECT nextval('demo_seq');  -- 13
```

Penjelasan:

- `nextval()` selalu mengambil dari **global sequence state**
- Tapi `currval()` hanya mengingat nilai terakhir di session tersebut

Jadi:

- Session A tetap melihat `10` saat memanggil `currval()`
- Tapi saat memanggil `nextval()` lagi, dia ikut melanjutkan dari global state (12 → 13)

#### Gambaran Praktis

Bayangkan sequence seperti dispenser nomor antrian di bank:

- Semua orang (session) ambil nomor dari mesin yang sama
- Nomor selalu naik, tidak peduli siapa yang mengambil
- Tapi setiap orang tetap ingat nomor terakhir yang dia ambil sendiri (`currval()`)

Dengan memahami ini, kamu bisa lebih percaya diri menggunakan sequence di aplikasi yang memiliki banyak user atau koneksi sekaligus, tanpa khawatir soal duplikasi ID atau konflik data.

### h. Use Cases untuk Manual Sequences

Meskipun banyak database sudah menyediakan auto-increment secara otomatis, ada beberapa skenario di mana menggunakan sequence secara manual justru lebih fleksibel dan powerful. Di bagian ini kita akan bahas beberapa use case nyata yang sering ditemui.

#### 1. Multiple Tables Sharing Sequence

Biasanya, setiap tabel punya sequence sendiri. Tapi dalam beberapa kasus, kamu mungkin ingin **beberapa tabel berbagi satu sequence yang sama**.

```sql id="3gk8m2"
CREATE SEQUENCE global_id_seq AS BIGINT;

CREATE TABLE orders (
    id BIGINT DEFAULT nextval('global_id_seq'),
    -- ...
);

CREATE TABLE invoices (
    id BIGINT DEFAULT nextval('global_id_seq'),
    -- ...
);
```

Penjelasan:

- Kedua tabel (`orders` dan `invoices`) menggunakan sequence yang sama
- Setiap kali ada insert ke salah satu tabel, ID akan diambil dari `global_id_seq`
- Hasilnya: **tidak akan ada ID yang sama antar tabel**

Ini berguna jika:

- Kamu ingin ID yang unik secara global (bukan hanya per tabel)
- Misalnya untuk sistem logging, audit trail, atau integrasi lintas tabel

#### 2. Custom Number Formatting

Sequence juga sering digunakan untuk membuat format nomor yang lebih “manusiawi”, seperti nomor invoice.

```sql id="q1z9vp"
CREATE SEQUENCE invoice_seq START WITH 1000;

CREATE TABLE invoices (
    id BIGINT PRIMARY KEY,
    invoice_number TEXT GENERATED ALWAYS AS
        ('INV-' || TO_CHAR(id, 'FM000000')) STORED
);
```

Penjelasan:

- `id` tetap berupa angka (misalnya 1000, 1001, dst)
- Kolom `invoice_number` menghasilkan format seperti:
  - `INV-001000`
  - `INV-001001`

- `TO_CHAR` digunakan untuk menambahkan padding nol di depan

Keuntungan:

- ID tetap efisien untuk database
- Tapi tetap punya format yang rapi untuk ditampilkan ke user

#### 3. Multiple Sequences untuk Partitioning

Dalam sistem yang besar, kadang kita ingin memisahkan data berdasarkan kategori tertentu, misalnya region.

```sql id="p7t4ls"
-- Sequence per region
CREATE SEQUENCE orders_us_seq;
CREATE SEQUENCE orders_eu_seq;
CREATE SEQUENCE orders_asia_seq;
```

Penjelasan:

- Setiap region punya sequence sendiri
- Order dari US, EU, dan Asia tidak saling berbagi counter

Manfaatnya:

- Bisa melacak urutan data per region
- Mengurangi contention jika traffic sangat tinggi
- Memberikan fleksibilitas dalam desain sistem terdistribusi

#### 4. Temporary Sequences untuk Migration

Saat melakukan migrasi data, sering kali kita butuh sequence sementara.

```sql id="m8x2qn"
-- Sequence sementara untuk data migration
CREATE SEQUENCE migration_seq START WITH 1000000;

-- Migrate data dengan ID di range tertentu
-- Bisa di-drop setelah selesai
DROP SEQUENCE migration_seq;
```

Penjelasan:

- Sequence dibuat khusus untuk proses migrasi
- Dimulai dari angka besar (misalnya 1.000.000) agar tidak bentrok dengan data lama
- Setelah selesai, sequence bisa dihapus

Use case ini umum saat:

- Import data dari sistem lama
- Menggabungkan beberapa database
- Melakukan re-indexing atau restructuring data

Dengan memahami berbagai use case ini, kamu bisa melihat bahwa sequence bukan hanya untuk auto-increment sederhana, tapi juga bisa menjadi alat yang sangat fleksibel dalam merancang sistem database yang kompleks dan scalable.

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
