# 📝 Covering Index dalam PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas konsep **covering index** dalam konteks database PostgreSQL. Setelah sebelumnya membahas berbagai jenis index (single-column index, composite index, dan bitmap index), kini diperkenalkan situasi khusus di mana query dapat **sepenuhnya dilayani oleh index itu sendiri** tanpa perlu mengakses heap (tabel asli). Topik ini mencakup cara mengenali covering index, cara membuatnya, serta batasan-batasan yang perlu dipahami.

---

## 2. Konsep Utama

### a. Alur Normal Penggunaan Index (Sebelum Covering Index)

Sebelum kita masuk ke konsep _covering index_, penting untuk memahami dulu bagaimana alur dasar penggunaan index di database bekerja. Ini berlaku untuk beberapa jenis index yang umum digunakan, seperti:

- **Single index** (index pada satu kolom)
- **Composite index** (index pada beberapa kolom sekaligus)
- **Dua single index** yang dikombinasikan (biasanya menggunakan teknik seperti bitmap)

Meskipun bentuk dan strukturnya berbeda, cara kerja ketiganya pada dasarnya sama. Mari kita bahas alurnya secara runtut.

#### Bagaimana Database Menggunakan Index

Ketika sebuah query dijalankan, database tidak langsung mengambil data dari tabel. Sebaliknya, ia memanfaatkan index terlebih dahulu untuk mempercepat pencarian.

Alurnya seperti ini:

1. **Database menelusuri index**
   Index berisi struktur yang sudah terurut (biasanya berbasis B-Tree), sehingga pencarian bisa dilakukan dengan cepat. Dari sini, database akan menemukan lokasi data yang dicari, bukan datanya langsung.

2. **Mendapatkan row address**
   Hasil dari pencarian di index adalah _row address_ (alamat baris), yaitu pointer yang menunjukkan di mana data tersebut berada di dalam tabel.

3. **Masuk ke heap (tabel utama)**
   Setelah mendapatkan alamatnya, database kemudian pergi ke heap — yaitu tempat penyimpanan data sebenarnya.

4. **Mengambil data dari heap**
   Database mengambil baris data yang dibutuhkan dari heap berdasarkan alamat tadi.

5. **Mengembalikan hasil ke pemanggil**
   Data yang sudah diambil kemudian dikirim sebagai hasil query.

Secara sederhana, alurnya bisa digambarkan seperti ini:

```
Query → Index → (dapat row address) → Heap → Ambil baris → Kembalikan hasil
```

#### Contoh Sederhana

Misalkan kita punya query seperti ini:

```sql
SELECT name FROM users WHERE email = 'user@example.com';
```

Dan terdapat index pada kolom `email`.

Yang terjadi adalah:

- Database mencari `'user@example.com'` di index.
- Index memberikan _row address_ (misalnya baris ke-125).
- Database pergi ke heap untuk mengambil data pada baris ke-125.
- Dari baris tersebut, diambil kolom `name`.
- Hasil dikembalikan ke pengguna.

#### Insight Penting

Di sini ada satu hal penting yang sering terlewat:
**index tidak menyimpan seluruh data**, melainkan hanya sebagian (biasanya kolom yang di-index + pointer ke data asli).

Artinya, meskipun index membantu menemukan lokasi data dengan cepat, database tetap perlu melakukan “perjalanan tambahan” ke heap untuk mengambil data sebenarnya.

Langkah tambahan inilah yang nantinya akan coba dihindari oleh konsep _covering index_, yang akan dibahas di bagian berikutnya.

### b. Apa Itu Covering Index?

Setelah memahami alur normal penggunaan index (yang masih perlu “bolak-balik” ke heap), sekarang kita masuk ke konsep yang lebih efisien: **covering index**.

Secara sederhana, **covering index** adalah kondisi di mana **semua kebutuhan query sudah bisa dipenuhi langsung dari index**, tanpa harus mengambil data tambahan dari tabel (heap).

#### Inti dari Covering Index

Sebuah query disebut menggunakan covering index jika seluruh komponen berikut tersedia di dalam index:

- Kolom yang diambil (`SELECT`)
- Kolom yang digunakan untuk filter (`WHERE`)
- Kolom untuk pengelompokan (`GROUP BY`)
- Kolom untuk pengurutan (`ORDER BY`)

Karena semua informasi sudah ada di index, database tidak perlu lagi melakukan langkah tambahan ke heap. Ini membuat query jadi jauh lebih cepat.

Alurnya berubah menjadi jauh lebih sederhana:

```
Query → Index → (semua data sudah ada) → Kembalikan hasil ✅
         ↑
    Tidak perlu ke Heap!
```

#### Hal Penting yang Perlu Dipahami

Covering index sering disalahpahami sebagai jenis index baru. Padahal:

- Ini **bukan tipe index baru**
- Ini adalah **index biasa** (misalnya composite index)
- Disebut “covering” karena **kebetulan memenuhi semua kebutuhan query**

Jadi, sifatnya kontekstual — tergantung bagaimana query ditulis dan bagaimana index dibuat.

#### Contoh Kasus

Misalkan kita punya tabel `users`, lalu kita buat index seperti ini:

```sql
CREATE INDEX ON users (first_name);
```

Index ini hanya menyimpan kolom `first_name` dan pointer ke data asli.

Sekarang kita lihat dua query berbeda.

#### Query yang Tidak Menggunakan Covering Index

```sql
EXPLAIN SELECT * FROM users WHERE first_name = 'Aaron';
```

Apa yang terjadi?

- Database menggunakan index untuk mencari `first_name = 'Aaron'`
- Tapi karena kita meminta `SELECT *`, database butuh semua kolom
- Index tidak punya semua kolom tersebut
- Akhirnya database tetap harus ke heap

Biasanya akan terlihat di hasil `EXPLAIN` seperti:

- `Bitmap Index Scan`
- `Bitmap Heap Scan`

Artinya: index dipakai, tapi tetap lanjut ke heap.

#### Query yang Menggunakan Covering Index

```sql
EXPLAIN SELECT first_name FROM users WHERE first_name = 'Aaron';
```

Di sini situasinya berbeda:

- Query hanya membutuhkan `first_name`
- Kolom itu sudah tersedia di index
- Tidak ada kebutuhan tambahan ke heap

Hasilnya, database cukup membaca dari index saja.

Biasanya ditunjukkan dengan:

- **`Index Only Scan` ✅**

#### Insight Penting

Perbedaan kecil dalam query bisa berdampak besar pada performa.

Mengubah dari:

```sql
SELECT *
```

menjadi:

```sql
SELECT first_name
```

bisa menghilangkan kebutuhan akses ke heap sepenuhnya.

Inilah kekuatan utama covering index:
**mengurangi I/O ke storage dengan memaksimalkan penggunaan index.**

Semakin sering query bisa diselesaikan hanya dari index, semakin cepat performa yang bisa kita dapatkan.

### c. Composite Index sebagai Covering Index

Salah satu cara paling efektif untuk memanfaatkan covering index adalah dengan menggunakan **composite index** (index yang berisi lebih dari satu kolom).

Kenapa? Karena semakin banyak kolom yang dimasukkan ke dalam index, semakin besar kemungkinan sebuah query bisa “ter-cover” sepenuhnya oleh index tersebut — tanpa perlu ke heap.

#### Konsep Dasarnya

Ketika kita membuat composite index, kita sebenarnya menyimpan beberapa kolom sekaligus di dalam struktur index. Ini berarti:

- Query yang hanya membutuhkan kolom-kolom tersebut bisa langsung dilayani dari index
- Database tidak perlu lagi mengambil data tambahan dari heap

Dengan kata lain, composite index memperluas peluang terjadinya covering index.

#### Contoh Penggunaan

Misalkan kita punya tabel `users`, lalu kita buat composite index seperti ini:

```sql id="c3k2df"
CREATE INDEX ON users (first_name, last_name);
```

Index ini sekarang menyimpan dua kolom: `first_name` dan `last_name`.

#### Query yang Menggunakan Covering Index

```sql id="m1z9qp"
EXPLAIN SELECT first_name, last_name FROM users WHERE first_name = 'Aaron';
```

Yang terjadi:

- Filter menggunakan `first_name` → tersedia di index
- Kolom yang diambil: `first_name`, `last_name` → keduanya ada di index
- Tidak ada kebutuhan data lain

Hasilnya:

- Database cukup membaca dari index
- Tidak perlu ke heap
- Di `EXPLAIN` akan muncul: **`Index Only Scan` ✅**

#### Tetap Efektif dengan ORDER BY

```sql id="n8x4rl"
EXPLAIN SELECT first_name, last_name
FROM users
WHERE first_name = 'Aaron'
ORDER BY last_name;
```

Di sini:

- `ORDER BY last_name` juga masih bisa dilayani dari index
- Karena `last_name` termasuk dalam composite index

Artinya:

- Sorting bisa dilakukan tanpa akses tambahan ke heap
- Tetap menggunakan **`Index Only Scan`**

Ini menunjukkan bahwa covering index tidak hanya berlaku untuk `SELECT` dan `WHERE`, tapi juga bisa mencakup `ORDER BY` selama kolomnya tersedia di index.

#### Ketika Covering Index Tidak Berlaku

Sekarang lihat contoh berikut:

```sql id="q7w2ht"
EXPLAIN SELECT first_name, last_name, email
FROM users
WHERE first_name = 'Aaron';
```

Di sini:

- `email` tidak ada di dalam index
- Database tidak punya pilihan selain mengambil data dari heap

Akibatnya:

- Index tetap digunakan untuk mencari baris
- Tapi harus lanjut ke heap untuk mengambil `email`
- Biasanya muncul:
  `Bitmap Index Scan + Bitmap Heap Scan`

#### Insight Penting

Composite index memberi fleksibilitas lebih besar untuk menciptakan covering index, tapi ada trade-off:

- Semakin banyak kolom di index → ukuran index makin besar
- Penulisan data (INSERT/UPDATE) bisa sedikit lebih lambat

Jadi, kuncinya bukan sekadar menambahkan banyak kolom, tapi **memilih kolom yang benar-benar sering digunakan bersama dalam query**.

Dengan desain yang tepat, composite index bisa menjadi alat yang sangat powerful untuk menghindari akses ke heap dan meningkatkan performa query secara signifikan.

### d. Fitur `INCLUDE` di PostgreSQL

Saat kita ingin membuat covering index, sering muncul dilema: kita butuh kolom tambahan agar query bisa sepenuhnya dilayani dari index, tapi kalau semua kolom dimasukkan ke dalam index utama (B-tree), ukuran dan kompleksitas index bisa meningkat.

Di sinilah PostgreSQL menawarkan solusi yang cukup elegan melalui fitur `INCLUDE`.

#### Apa Itu `INCLUDE`?

Fitur `INCLUDE` memungkinkan kita **menambahkan kolom ke dalam index tanpa menjadikannya bagian dari struktur pencarian (B-tree)**.

Artinya:

- Kolom utama (yang ada di dalam tanda kurung biasa) tetap digunakan untuk navigasi index
- Kolom tambahan (yang ada di `INCLUDE`) hanya “menempel” di bagian akhir index, tepatnya di **leaf node**

#### Cara Kerja Secara Konseptual

Kolom yang di-`INCLUDE` memiliki karakteristik berikut:

- **Tidak digunakan untuk traversal B-tree**
  Database tidak akan menggunakan kolom ini untuk mencari atau menyaring data.

- **Hanya disimpan di leaf node**
  Kolom ini tetap tersedia sebagai data tambahan saat hasil index dibaca.

- **Berfungsi untuk mendukung covering index**
  Tujuannya agar query yang membutuhkan kolom tambahan tetap bisa dilayani tanpa ke heap.

Dengan kata lain, `INCLUDE` memungkinkan kita menambah “payload” ke index tanpa mengganggu struktur utamanya.

#### Perbandingan dengan Cara Biasa

Tanpa `INCLUDE`, jika kita ingin membawa kolom tambahan ke dalam index, kita harus memasukkannya langsung ke dalam B-tree:

```sql id="f2k8ls"
CREATE INDEX ON users (first_name, last_name, id);
```

Di sini, `id` menjadi bagian dari struktur index. Ini berarti:

- Ukuran index bertambah
- Struktur B-tree ikut berubah
- Bisa berdampak pada performa operasi tertentu

Dengan `INCLUDE`, kita bisa menulis seperti ini:

```sql id="u9x3ad"
CREATE INDEX multi ON users (first_name, last_name) INCLUDE (id);
```

Perbedaannya:

- `first_name` dan `last_name` → digunakan untuk navigasi index
- `id` → hanya disimpan sebagai data tambahan di leaf node

#### Dampak pada Query

Sekarang perhatikan query berikut:

```sql id="z7q1wr"
EXPLAIN SELECT first_name, last_name, id
FROM users
WHERE first_name = 'Aaron';
```

Yang terjadi:

- Filter menggunakan `first_name` → tersedia di index utama
- Kolom yang diambil: `first_name`, `last_name`, dan `id`
- Meskipun `id` bukan bagian dari B-tree, nilainya tetap tersedia di leaf node

Hasilnya:

- Database tidak perlu ke heap
- Query bisa dilayani sepenuhnya dari index
- Di `EXPLAIN` akan muncul: **`Index Only Scan` ✅**

#### Gambaran Struktur Sederhana

Secara konseptual, struktur index dengan `INCLUDE` bisa dibayangkan seperti ini:

```
[B-Tree Structure]
  first_name
      ↓
  last_name
      ↓
[Leaf Node]
  first_name, last_name
  + data tambahan: id
```

Perhatikan bahwa `id` tidak ikut dalam jalur pencarian (tidak dilalui saat traversal), tapi tetap tersedia saat data diambil.

#### Insight Penting

Fitur `INCLUDE` memberi fleksibilitas dalam desain index:

- Kita bisa menjaga B-tree tetap ramping (hanya kolom penting untuk filter/sort)
- Tapi tetap mendukung covering index dengan menambahkan kolom ekstra

Ini sangat berguna untuk kasus di mana:

- Kolom sering di-_SELECT_
- Tapi jarang atau tidak pernah digunakan untuk filter (`WHERE`) atau sorting (`ORDER BY`)

Dengan pendekatan ini, kita bisa mendapatkan performa optimal tanpa “membebani” struktur index utama secara berlebihan.

### e. Kapan Covering Index Tidak Berfungsi Optimal? (Visibility Map)

Secara teori, covering index memungkinkan query berjalan tanpa perlu menyentuh heap sama sekali. Namun, di PostgreSQL ada satu faktor penting yang bisa membuat performanya tidak selalu maksimal, yaitu mekanisme **MVCC (Multi-Version Concurrency Control)**.

#### Kenapa Masih Perlu Pengecekan Tambahan?

PostgreSQL tidak hanya peduli _di mana data berada_, tapi juga _apakah data tersebut valid untuk transaksi yang sedang berjalan_.

Karena adanya MVCC, satu baris data bisa memiliki beberapa versi (misalnya karena UPDATE atau DELETE). Jadi, meskipun index sudah menemukan baris yang tepat, database tetap harus memastikan bahwa baris tersebut:

- Masih berlaku (belum dihapus)
- Terlihat oleh transaksi saat ini

Untuk itulah PostgreSQL menggunakan **visibility map**.

#### Peran Visibility Map

Visibility map adalah struktur tambahan yang membantu PostgreSQL mengetahui apakah satu halaman data di heap:

- Sudah “aman” (semua baris di dalamnya valid dan visible)
- Atau masih perlu dicek ulang ke heap

Dengan bantuan ini, PostgreSQL bisa memutuskan apakah benar-benar bisa menghindari akses ke heap, atau tidak.

#### Alur yang Terjadi

Berikut alur saat query menggunakan covering index:

1. PostgreSQL membaca index dan mendapatkan **row identifier (row ID)**
2. PostgreSQL mengecek **visibility map**
3. Berdasarkan hasilnya:
   - Jika valid → langsung gunakan hasil dari index
   - Jika tidak → harus ke heap untuk verifikasi

Secara sederhana:

```id="1r8qsm"
Covering Index → Dapat row IDs → Cek Visibility Map
                                   ↓
                       [Tabel jarang diubah]   [Tabel sering diubah]
                                   ↓                    ↓
                           Tidak ke heap         Tetap ke heap
                        (Index Only Scan)     (manfaat berkurang)
```

#### Dua Skenario yang Berbeda

##### 1. Tabel Jarang Berubah

Jika tabel:

- Jarang mengalami `INSERT`, `UPDATE`, atau `DELETE`
- Sudah di-_vacuum_ dengan baik

Maka:

- Visibility map cenderung valid
- PostgreSQL yakin bahwa data di index bisa langsung digunakan
- **Index Only Scan berjalan optimal (tanpa ke heap)**

##### 2. Tabel Sering Berubah

Jika tabel:

- Sering mengalami perubahan data
- Banyak versi baris yang belum dibersihkan

Maka:

- Visibility map sering tidak valid
- PostgreSQL tidak bisa langsung percaya hasil dari index
- Harus tetap ke heap untuk memastikan visibilitas

Akibatnya:

- Masih ada akses ke heap
- Keuntungan covering index jadi berkurang

#### Insight Penting

Walaupun tetap perlu pengecekan visibility map, proses ini **jauh lebih ringan dibanding langsung membaca heap**. Jadi, covering index tetap memberikan keuntungan — hanya saja tidak selalu “sempurna” dalam semua kondisi.

Intinya:

- Covering index paling optimal pada data yang relatif stabil
- Pada data yang sangat dinamis, manfaatnya masih ada, tapi tidak maksimal

Memahami aspek ini penting agar kita tidak hanya fokus pada desain index, tapi juga pada **karakteristik data dan pola perubahan tabel**.

### f. Trade-off dan Batasan

Covering index memang powerful, tapi bukan berarti selalu jadi solusi terbaik di semua situasi. Ada beberapa trade-off yang perlu dipahami supaya kita bisa menggunakannya dengan tepat, bukan sekadar “mengejar performa” tanpa mempertimbangkan dampaknya.

#### ❌ Mengapa tidak selalu pakai covering index?

##### 1. `SELECT *` menghancurkan peluang covering index

Ketika kita menulis query seperti:

```sql id="a1x2bz"
SELECT * FROM users WHERE first_name = 'Aaron';
```

Artinya kita meminta **semua kolom** dari tabel. Masalahnya:

- Index biasanya hanya menyimpan beberapa kolom saja
- Hampir pasti ada kolom yang tidak tersedia di index

Akibatnya:

- Database tetap harus pergi ke heap
- Covering index tidak bisa digunakan

Ini menguatkan prinsip penting dalam query design:
**ambil hanya kolom yang benar-benar dibutuhkan (_select only what you need_)**

##### 2. Terlalu banyak kolom membuat index membengkak

Misalnya kita mencoba “memaksa” covering index dengan memasukkan banyak kolom:

```sql id="k9lm3p"
CREATE INDEX ON users (first_name, last_name, email, address, profile_json);
```

Kelihatannya membantu, tapi ada konsekuensi:

- Ukuran index jadi besar (terutama jika ada kolom seperti `TEXT` atau `JSON`)
- Struktur B-tree jadi lebih berat
- Operasi seperti `INSERT` dan `UPDATE` jadi lebih lambat karena index harus ikut diperbarui

Dalam kasus ekstrem, ukuran index bisa mendekati ukuran tabel itu sendiri — yang jelas bukan efisien.

##### 3. Tabel yang sering berubah mengurangi efektivitas

Seperti yang sudah dibahas sebelumnya, PostgreSQL menggunakan **visibility map** untuk memastikan data valid.

Jika tabel:

- Sering mengalami `INSERT`, `UPDATE`, atau `DELETE`

Maka:

- Visibility map sering tidak valid
- Database tetap harus mengecek heap

Artinya, meskipun secara teori sudah covering index, praktiknya masih ada akses ke heap, sehingga manfaatnya berkurang.

#### ✅ Kapan covering index paling bermanfaat?

Sekarang kita lihat kapan strategi ini benar-benar memberikan hasil optimal.

##### 1. Tabel yang jarang berubah

Jika data relatif statis:

- Visibility map cenderung valid
- Query bisa benar-benar berjalan dengan **Index Only Scan**
- Tidak perlu akses ke heap

Contohnya: tabel referensi, data katalog, atau lookup table.

##### 2. Query yang sering dijalankan (hot path)

Untuk query yang:

- Sering dieksekusi
- Menjadi jalur utama aplikasi (_critical path_)

Mengoptimalkan dengan covering index bisa memberikan dampak signifikan terhadap performa keseluruhan sistem.

##### 3. Query yang hanya mengambil beberapa kolom

Misalnya:

```sql id="w4e7yn"
SELECT first_name, last_name
FROM users
WHERE first_name = 'Aaron';
```

Karena:

- Kolom yang diambil sedikit
- Mudah dimasukkan ke dalam index

Maka peluang menjadi covering index jauh lebih besar.

#### Insight Penutup

Covering index bukan soal “menambahkan semua kolom ke index”, tapi soal **menemukan keseimbangan**:

- Cukup kolom untuk memenuhi query penting
- Tapi tidak berlebihan hingga membebani sistem

Dengan memahami trade-off ini, kita bisa merancang index yang benar-benar efektif — bukan hanya cepat di satu sisi, tapi juga tetap efisien dalam jangka panjang.

## 3. Hubungan Antar Konsep

```
Index Biasa (Single / Composite / Bitmap)
        ↓
  Semua jenis ini → Cari row address di index → Ke heap

Covering Index (situasi khusus dari index di atas)
        ↓
  Semua kebutuhan query ada di index → Tidak perlu ke heap
        ↓
  Terdeteksi dengan: "Index Only Scan" di output EXPLAIN
        ↓
  Bisa diperluas dengan fitur INCLUDE (PostgreSQL-specific)
        ↓
  Efektivitasnya dipengaruhi oleh: Visibility Map + frekuensi perubahan tabel
```

Secara keseluruhan, covering index adalah **puncak optimasi index**: ketika sebuah index biasa berada dalam kondisi ideal di mana semua kebutuhan query sudah tersedia di dalamnya. Fitur `INCLUDE` di PostgreSQL mempermudah mencapai kondisi ini tanpa membengkakkan struktur B-tree secara tidak perlu.

---

## 4. Kesimpulan

- **Covering index** bukanlah jenis index baru, melainkan situasi di mana index yang ada mampu memenuhi **seluruh kebutuhan query** tanpa perlu mengakses heap.
- Dapat dideteksi dengan melihat **`Index Only Scan`** pada output `EXPLAIN`.
- PostgreSQL menyediakan fitur **`INCLUDE`** untuk menyertakan kolom ekstra di leaf node B-tree tanpa memasukkannya ke dalam struktur pencarian, sehingga mempermudah pembuatan covering index.
- Efektivitas covering index dipengaruhi oleh **visibility map**: tabel yang sering berubah dapat mengurangi manfaatnya karena PostgreSQL tetap perlu mengecek heap.
- **Praktik terbaik:** Gunakan covering index untuk query spesifik yang sering dijalankan, hindari `SELECT *`, dan jangan memasukkan kolom besar ke dalam index hanya demi mengejar covering index.
- Secara keseluruhan, covering index adalah **opsi terbaik** dalam skenario yang tepat karena menghindarkan akses ke heap sepenuhnya, memberikan peningkatan performa yang signifikan.
