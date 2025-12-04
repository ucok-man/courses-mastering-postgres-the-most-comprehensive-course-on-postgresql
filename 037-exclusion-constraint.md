# 📝 EXCLUSION Constraint di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas EXCLUSION constraint di PostgreSQL, sebuah constraint yang lebih powerful dan fleksibel dibandingkan UNIQUE constraint. EXCLUSION constraint memungkinkan kita untuk mencegah konflik data berdasarkan kriteria custom yang kompleks, seperti mencegah overlapping time ranges. Materi ini mencakup penggunaan GIST index, operator overlap, composite exclusion, extension btree_gist, dan partial exclusion constraint.

## 2. Konsep Utama

### a. Apa itu EXCLUSION Constraint?

#### Definisi

EXCLUSION constraint adalah fitur PostgreSQL yang mirip dengan UNIQUE, tetapi jauh lebih fleksibel. Jika UNIQUE hanya memastikan bahwa sebuah nilai tidak boleh sama persis dengan nilai lain dalam kolom tertentu, EXCLUSION memungkinkan Anda mendefinisikan aturan kustom tentang data mana yang dianggap “bertentangan.” Artinya, Anda bisa membuat logika sendiri mengenai kondisi seperti apa yang tidak boleh terjadi bersamaan.

Walaupun tidak sesering digunakan, EXCLUSION sangat berguna untuk kasus-kasus di mana aturan integritas data tidak cukup diwakili oleh sekadar pengecekan nilai yang identik.

#### Perbandingan dengan UNIQUE

Untuk memahami perbedaannya, perhatikan dua contoh berikut:

```sql
-- UNIQUE constraint: Mencegah duplikasi nilai yang sama persis
CREATE TABLE users (
    email TEXT UNIQUE  -- Tidak boleh ada dua baris dengan email yang sama
);
```

Pada contoh di atas, aturan UNIQUE memastikan bahwa setiap email harus benar-benar unik. PostgreSQL hanya memeriksa equality (`=`). Jika nilainya sama, maka baris baru ditolak.

Sekarang bandingkan dengan EXCLUSION:

```sql
-- EXCLUSION constraint: Mencegah konflik berdasarkan kondisi custom
CREATE TABLE reservations (
    reservation_period TSRANGE,
    EXCLUDE USING gist (reservation_period WITH &&)
    -- Tidak boleh ada dua reservasi yang periodenya saling overlap
);
```

Di sini, kita tidak sedang mencegah nilai yang identik. Justru, dua periode reservasi boleh berbeda total, tetapi **tidak boleh saling overlap**. Operator `&&` adalah operator “overlap” pada tipe range. Ketika PostgreSQL mendeteksi dua baris yang periodenya beririsan, ia akan menolak insert tersebut.

Dengan kata lain:

- UNIQUE hanya melihat apakah nilai A == nilai B.
- EXCLUSION memungkinkan Anda mengatakan: “Jangan izinkan kondisi X terhadap operator Y,” misalnya overlap, intersect, inside, dan sebagainya.

#### Kapan Menggunakan EXCLUSION

EXCLUSION constraint sangat tepat ketika integritas data Anda melibatkan aturan yang tidak bisa ditangkap hanya dengan equality. Beberapa contoh penggunaannya antara lain:

- Saat ingin mencegah **overlapping ranges**, seperti jadwal reservasi, booking ruangan, jadwal dokter, blok waktu kalender, atau periode validitas.
- Ketika aturan konflik ditentukan oleh **operator lain** selain `=` — misalnya `&&` (overlap), `<@` (is contained), atau operator geospasial.
- Saat membutuhkan logika validasi yang lebih kompleks daripada sekadar “nilai harus berbeda.” Misalnya, Anda hanya ingin mencegah overlap untuk tipe tertentu, tetapi mengizinkan overlap untuk tipe lainnya (diatur lewat kombinasi beberapa kolom dalam EXCLUSION).

Dengan EXCLUSION constraint, PostgreSQL memberi Anda cara yang sangat kuat untuk memastikan integritas data secara deklaratif, tanpa harus menulis trigger atau logic di level aplikasi.

### b. Struktur Dasar EXCLUSION Constraint

#### Sintaks

Secara umum, EXCLUSION constraint ditulis dengan pola berikut:

```sql
EXCLUDE USING index_method (
    column_name WITH operator,
    ...
)
```

Format ini memberi tahu PostgreSQL bahwa Anda ingin membuat aturan yang mencegah baris-baris tertentu memiliki hubungan tertentu terhadap operator tertentu. Berbeda dengan UNIQUE, yang hanya bekerja dengan equality, EXCLUSION memungkinkan Anda menentukan sendiri jenis konflik yang harus dicegah.

#### Komponen

Untuk memahami sintaksnya, mari kita lihat fungsi dari setiap bagian:

1. **EXCLUDE**
   Menandakan bahwa Anda sedang membuat exclusion constraint, yaitu aturan untuk mencegah kondisi tertentu muncul dalam tabel.

2. **USING gist**
   Menentukan jenis indeks yang digunakan untuk melakukan pemeriksaan.
   Kebanyakan EXCLUSION menggunakan **GiST index**, karena indeks ini mendukung berbagai operator seperti overlap, containment, atau operator geospasial. Topik ini akan dibahas lebih detail pada modul indexing.

3. **column_name**
   Kolom yang akan menjadi dasar pengecekan konflik. Anda dapat menempatkan satu atau beberapa kolom sekaligus.

4. **WITH operator**
   Ini bagian yang paling penting — operator inilah yang menentukan kondisi apa yang dianggap bertentangan. Misalnya `&&` untuk overlap pada range.

Dengan komposisi ini, Anda bisa membuat constraint yang jauh lebih fleksibel dibandingkan sekadar UNIQUE.

#### Contoh Paling Sederhana

Mari kita lihat contoh implementasi nyata:

```sql
CREATE TABLE reservations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INTEGER,
    reservation_period TSRANGE,
    EXCLUDE USING gist (reservation_period WITH &&)
);
```

Pada tabel ini, kita menyimpan data reservasi. Setiap reservasi memiliki `reservation_period` bertipe `TSRANGE`, yang mewakili rentang waktu mulai hingga selesai.

EXCLUSION constraint `reservation_period WITH &&` artinya:

- PostgreSQL akan mengecek apakah rentang waktu dari baris baru **beririsan** dengan rentang waktu pada baris lain.
- Jika ada dua reservasi dengan waktu yang overlap, maka baris baru tersebut akan ditolak.

#### Penjelasan Mekanisme

Saat Anda melakukan `INSERT` atau `UPDATE`, PostgreSQL menjalankan langkah-langkah berikut:

1. Mengambil nilai `reservation_period` dari baris yang baru.
2. Membandingkan nilai tersebut dengan seluruh baris yang sudah ada di tabel.
3. Menggunakan operator `&&` untuk mengecek apakah kedua rentang waktu saling overlap.
4. Jika ditemukan konflik (ada overlap), PostgreSQL langsung menolak operasi tersebut dan mengembalikan error.

Dengan cara ini, EXCLUSION constraint memberi jaminan bahwa tidak akan pernah ada dua reservasi yang menggunakan rentang waktu yang saling bertabrakan. Ini sangat berguna dalam sistem booking, jadwal, dan segala bentuk data yang berhubungan dengan range.

### c. Operator Overlap (`&&`) untuk Range Types

#### Apa itu operator `&&`?

Operator `&&` adalah operator khusus di PostgreSQL yang digunakan untuk mengecek apakah dua **range** saling overlap. Range di sini bisa berupa rentang waktu (TSRANGE), tanggal (DATERANGE), angka (INT4RANGE), dan berbagai tipe range lainnya.
Jika dua range memiliki irisan, sekecil apa pun, operator `&&` akan mengembalikan nilai **true**.

Operator inilah yang menjadi jantung dari banyak EXCLUSION constraint, terutama untuk sistem reservasi, kalender, atau penjadwalan.

#### Contoh Penggunaan

Perhatikan contoh berikut untuk memahami cara kerja overlap dalam konteks reservasi:

```sql
-- Insert pertama berhasil
INSERT INTO reservations (room_id, reservation_period)
VALUES (1, '[2024-09-01 12:00, 2024-09-03 12:00)');
-- ✅ Berhasil
```

Insert pertama sukses karena tabel masih kosong. Kita sekarang punya reservasi mulai 1 September pukul 12:00 sampai 3 September pukul 12:00.

Selanjutnya, kita mencoba menambahkan reservasi lain:

```sql
-- Data yang ada: 01 Sep - 03 Sep
-- Coba insert: 02 Sep - 04 Sep
INSERT INTO reservations (room_id, reservation_period)
VALUES (1, '[2024-09-02 12:00, 2024-09-04 12:00)');
-- ❌ Error: conflicting key value excludes reservation
```

Reservasi kedua gagal. Penyebabnya adalah operator `&&` mendeteksi bahwa periode 02–04 September **beririsan** dengan periode 01–03 September. Karena EXCLUSION constraint mencegah overlap, PostgreSQL menolak baris tersebut.

#### Visualisasi Overlap

Untuk memperjelas bagaimana overlap terjadi, berikut gambaran rentang waktunya:

```
Existing: [01 Sep -------- 03 Sep)
New:              [02 Sep -------- 04 Sep)
                   ↑ Overlap di sini!
```

Kedua rentang ini jelas saling bersinggungan. Itulah mengapa operasi insert kedua menghasilkan error.

#### Catatan Penting tentang Range Bounds

PostgreSQL range types menggunakan tanda kurung untuk menentukan apakah batasnya termasuk atau tidak:

- `[2024-09-01 12:00, 2024-09-03 12:00)` berarti:

  - `[` → **inclusive**, batas awal termasuk (1 Sep 12:00 masuk dalam range)
  - `)` → **exclusive**, batas akhir tidak termasuk (3 Sep 12:00 dianggap waktu checkout, jadi tidak ikut dihitung sebagai bagian dari rentang)

Konvensi ini sangat penting dalam sistem reservasi karena menentukan apakah dua periode dianggap overlap atau tidak. Misalnya, jika satu reservasi berakhir tepat pada batas exclusive waktu berikutnya, PostgreSQL menganggap keduanya **tidak overlap**.

### d. Masalah dengan EXCLUSION Sederhana

#### Problem: Terlalu Ketat untuk Skenario Multi-Room

Contoh berikut menunjukkan bagaimana EXCLUSION constraint yang terlalu sederhana dapat menimbulkan masalah ketika digunakan dalam sistem yang memiliki lebih dari satu ruangan.

```sql
CREATE TABLE reservations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INTEGER,
    reservation_period TSRANGE,
    EXCLUDE USING gist (reservation_period WITH &&)
);

-- Room 1: 01-03 Sep
INSERT INTO reservations VALUES (DEFAULT, 1, '[2024-09-01, 2024-09-03)');
-- ✅ Berhasil
```

Pada tahap ini, semuanya berjalan normal. Kita memiliki satu reservasi untuk Room 1 antara tanggal 1–3 September.

Namun perhatikan ketika kita mencoba memasukkan reservasi kedua untuk ruangan lain:

```sql
-- Room 2: 01-03 Sep (harusnya boleh, karena room berbeda!)
INSERT INTO reservations VALUES (DEFAULT, 2, '[2024-09-01, 2024-09-03)');
-- ❌ Error: Ditolak! (padahal room berbeda)
```

Padahal kedua ruangan berbeda, sehingga seharusnya tidak ada konflik. Room 1 dan Room 2 tidak saling bergantung, sehingga memiliki reservasi di waktu yang sama seharusnya diperbolehkan.

#### Mengapa Ini Terjadi?

Masalahnya terletak pada definisi constraint yang terlalu sederhana. EXCLUSION constraint di atas hanya mengecek **reservation_period**, tanpa mempertimbangkan **room_id**. Dengan kata lain:

- PostgreSQL hanya melihat apakah periode waktu baru overlap dengan periode waktu yang sudah ada.
- Ia tidak mengecek apakah overlap tersebut terjadi pada ruangan yang sama atau berbeda.
- Akibatnya, semua overlap dianggap konflik, meskipun data tersebut sebenarnya valid dalam konteks multi-room.

EXCLUSION constraint bekerja persis seperti yang kita definisikan — dan definisi saat ini belum lengkap.

#### Solusi: Sertakan `room_id` dalam EXCLUSION Constraint

Untuk membuat sistem reservasi multi-room yang benar, kita perlu memberi tahu database bahwa **overlap hanya dianggap konflik jika terjadi pada ruangan yang sama**.
Solusinya adalah menambahkan `room_id` ke dalam EXCLUSION constraint, biasanya dengan operator `=` untuk menunjukkan bahwa hanya baris dengan room_id yang sama yang perlu dibandingkan.

Dengan langkah ini, PostgreSQL akan:

1. Mengecek apakah `room_id` sama.
2. Jika sama, barulah ia mengevaluasi apakah `reservation_period` overlap.
3. Jika berbeda, maka overlap diabaikan — sesuai yang kita inginkan.

Pada bagian berikutnya, constraint yang lebih lengkap akan membuat sistem reservasi bekerja dengan benar untuk banyak ruangan.

### e. Composite EXCLUSION Constraint

#### Solusi: Periksa Kombinasi `room_id` dan `reservation_period`

Untuk menghindari masalah pada contoh sebelumnya, kita perlu memberi tahu database bahwa overlap hanya dianggap konflik jika terjadi pada ruangan yang sama. Caranya adalah membuat composite EXCLUSION constraint yang mengevaluasi dua hal sekaligus: apakah `room_id` sama, dan apakah `reservation_period` overlap.

Contoh implementasinya:

```sql
CREATE TABLE reservations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INTEGER,
    reservation_period TSRANGE,
    EXCLUDE USING gist (
        room_id WITH =,              -- Room harus sama DAN
        reservation_period WITH &&   -- Periode overlap
    )
);
```

Dengan definisi ini, kita menginstruksikan PostgreSQL untuk menerapkan aturan konflik berdasarkan **kombinasi dua kondisi**, bukan salah satunya saja.

#### Logika

Constraint ini hanya akan menolak baris baru jika **kedua kondisi berikut terpenuhi secara bersamaan**:

1. `room_id` sama persis (`=`)
2. `reservation_period` overlap (`&&`)

Jika salah satu dari kondisi tersebut tidak terpenuhi, maka baris dianggap valid.
Contohnya, dua reservasi dengan waktu yang beririsan tetap diperbolehkan selama mereka berada pada ruangan yang berbeda.

#### Contoh Penggunaan

Mari kita lihat bagaimana constraint ini bekerja dalam situasi nyata:

```sql
-- Room 1: 02-04 Sep
INSERT INTO reservations VALUES (DEFAULT, 1, '[2024-09-02, 2024-09-04)');
-- ✅ Berhasil
```

Reservasi pertama untuk Room 1 berhasil dimasukkan.

```sql
-- Room 2: 02-04 Sep (room berbeda, boleh!)
INSERT INTO reservations VALUES (DEFAULT, 2, '[2024-09-02, 2024-09-04)');
-- ✅ Berhasil
```

Meskipun periode waktunya sama persis, PostgreSQL tetap mengizinkan insert ini karena ruangan berbeda. Constraint bekerja sesuai desain: tidak ada konflik.

```sql
-- Room 2: 03-05 Sep (room sama dengan row sebelumnya, overlap!)
INSERT INTO reservations VALUES (DEFAULT, 2, '[2024-09-03, 2024-09-05)');
-- ❌ Error: Key (room_id, reservation_period)=(2, ...) conflicts with existing key
```

Kali ini PostgreSQL menolak insert karena:

- `room_id` sama (keduanya Room 2)
- Rentang waktu saling overlap (03–05 beririsan dengan 02–04)

Kedua syarat konflik terpenuhi, sehingga operasi diblokir.

#### Visualisasi

Berikut adalah gambaran sederhana untuk menunjukkan bagaimana constraint ini bekerja:

```
Room 1: [02 Sep -------- 04 Sep)          ✅
Room 2: [02 Sep -------- 04 Sep)          ✅ OK karena room berbeda
Room 2:        [03 Sep -------- 05 Sep)   ❌ Conflict! (room sama + overlap)
```

Dengan composite EXCLUSION constraint ini, sistem reservasi menjadi jauh lebih akurat: setiap ruangan memiliki aturan validasinya sendiri, dan overlap hanya dicegah ketika benar-benar relevan.

### f. Extension btree_gist

#### Problem dengan operator `=` pada GIST index

Ketika kita mencoba membuat composite EXCLUSION constraint menggunakan GIST index dan menyertakan operator `=` untuk kolom `room_id`, PostgreSQL akan menolak permintaan tersebut.

Contohnya:

```sql
EXCLUDE USING gist (
    room_id WITH =,  -- ❌ Error!
    reservation_period WITH &&
)
```

Jika kode ini dijalankan, PostgreSQL mengeluarkan error seperti berikut:

```
ERROR: data type integer has no default operator class for access method "gist"
```

#### Penjelasan Mengapa Terjadi Error

Pesan error ini muncul karena:

- GIST index **tidak memiliki operator class bawaan** untuk equality (`=`), terutama untuk tipe data sederhana seperti integer.
- GIST dirancang untuk data yang lebih kompleks — seperti data spasial, geometri, dan range — tempat operasi seperti overlap (`&&`) lebih relevan.
- Sebaliknya, B-tree index adalah struktur yang ideal untuk operasi equality karena ia selalu mengurutkan data secara konsisten.

Dengan kata lain, kita sedang meminta GIST melakukan sesuatu yang tidak ia dukung secara default, yakni mengevaluasi operator `=` untuk integer.

Untuk membuat EXCLUSION constraint yang menggabungkan operator equality (`=`) dan operator overlap (`&&`), kita membutuhkan tambahan fungsionalitas yang membuat GIST bisa memahami equality.

#### Solusi: Enable extension **btree_gist**

PostgreSQL menyediakan extension bernama **btree_gist** untuk mengatasi limitasi ini.

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- Sekarang constraint bisa dibuat
CREATE TABLE reservations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INTEGER,
    reservation_period TSRANGE,
    EXCLUDE USING gist (
        room_id WITH =,              -- ✅ Sekarang bisa!
        reservation_period WITH &&
    )
);
```

Setelah extension ini aktif, PostgreSQL menambahkan operator class berbasis B-tree ke dalam GIST index. Hasilnya, GIST kini dapat menangani operator `=` untuk tipe data yang sebelumnya tidak didukung — termasuk integer.

#### Apa yang dilakukan btree_gist?

Extension ini memungkinkan GIST index mengerti operator dan perilaku yang biasanya hanya dimiliki oleh B-tree. Secara sederhana:

- Ia **menambahkan kemampuan equality** (`=`) ke dalam GIST.
- Ia memungkinkan kita membuat EXCLUSION constraint yang mencampur operator equality dan operator overlap.
- Ia menyatukan keunggulan dua dunia:

  - B-tree → performa bagus untuk pencocokan nilai exact
  - GIST → kemampuan untuk range dan struktur data kompleks

Tanpa extension ini, kita tidak bisa membuat constraint yang memadukan `room_id WITH =` dan `reservation_period WITH &&`.

#### Catatan

Extension dan operator class adalah topik tersendiri dan akan dibahas lebih lengkap di modul lain.
Untuk sekarang, cukup pahami bahwa **btree_gist wajib** jika Anda ingin membuat EXCLUSION constraint yang menggunakan operator equality pada GIST index.

### g. Partial EXCLUSION Constraint (dengan WHERE)

#### Scenario real-world: Handling canceled reservations

Pada sistem reservasi nyata, sering kali ada dua jenis reservasi: yang masih aktif dan yang sudah dibatalkan. Ketika pengguna membatalkan reservasi, slot waktunya seharusnya kembali tersedia untuk dibooking ulang. Namun dengan EXCLUSION constraint yang kita buat sebelumnya, bagian ini tidak berjalan dengan baik.

Perhatikan contoh berikut:

```sql
CREATE TABLE reservations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INTEGER,
    reservation_period TSRANGE,
    booking_status TEXT,  -- 'confirmed', 'canceled', etc.
    EXCLUDE USING gist (
        room_id WITH =,
        reservation_period WITH &&
    )
);

-- Insert canceled reservation
INSERT INTO reservations
VALUES (DEFAULT, 1, '[2024-09-01, 2024-09-03)', 'canceled');
-- ✅ Berhasil

-- Coba book ulang dengan status confirmed (harusnya boleh!)
INSERT INTO reservations
VALUES (DEFAULT, 1, '[2024-09-01, 2024-09-03)', 'confirmed');
-- ❌ Error: Ditolak! (padahal yang pertama canceled)
```

Pada kasus ini, PostgreSQL tetap mendeteksi konflik karena EXCLUSION constraint tidak mempertimbangkan `booking_status`. Semua row, termasuk yang sudah dibatalkan, tetap dihitung dalam pengecekan overlap.

### Solusi: Tambahkan predicate dengan WHERE clause

Untuk memperbaiki logika ini, kita dapat membuat _partial exclusion constraint_ — yaitu exclusion constraint yang hanya diterapkan pada subset data tertentu. Caranya adalah dengan menambahkan kondisi `WHERE` pada definisi constraint.

```sql
CREATE TABLE reservations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INTEGER,
    reservation_period TSRANGE,
    booking_status TEXT,
    EXCLUDE USING gist (
        room_id WITH =,
        reservation_period WITH &&
    )
    WHERE (booking_status != 'canceled')  -- Predicate!
);
```

Dengan penambahan predicate ini, constraint hanya akan mengevaluasi baris-baris yang statusnya **bukan** `canceled`.

### Cara kerja

- Row dengan `booking_status != 'canceled'` → **diperiksa** oleh constraint.
- Row dengan `booking_status = 'canceled'` → **diabaikan**, diperlakukan seperti data yang tidak “aktif”.

Hasilnya, reservasi yang dibatalkan tidak menghalangi orang lain untuk membooking ulang slot yang sama.

### Contoh penggunaan

```sql
-- Insert canceled reservation
INSERT INTO reservations
VALUES (DEFAULT, 1, '[2024-09-01, 2024-09-03)', 'canceled');
-- ✅ Berhasil

-- Book ulang dengan confirmed (sekarang boleh!)
INSERT INTO reservations
VALUES (DEFAULT, 1, '[2024-09-01, 2024-09-03)', 'confirmed');
-- ✅ Berhasil! (canceled diabaikan dari constraint)

-- Coba insert confirmed kedua (conflict!)
INSERT INTO reservations
VALUES (DEFAULT, 1, '[2024-09-01, 2024-09-03)', 'confirmed');
-- ❌ Error: Tidak boleh 2 confirmed dengan room dan periode yang sama
```

Dengan partial constraint ini, PostgreSQL hanya melarang konflik antar reservasi yang statusnya aktif (confirmed, pending, atau yang lainnya — selama bukan `canceled`).

### Visualisasi

```
Constraint hanya cek row dengan booking_status != 'canceled'

Row 1: canceled  → Diabaikan (tidak dicek)
Row 2: confirmed → Dicek ✅
Row 3: confirmed → Dicek dan conflict dengan Row 2 ❌
```

### Business logic yang dituju

- Reservasi yang dibatalkan **mengembalikan slot menjadi available**.
- Sistem boleh menerima reservasi baru di slot waktu yang sama.
- Tetapi **tetap tidak boleh** ada lebih dari satu reservasi aktif yang overlapping untuk ruangan yang sama.

Dengan pendekatan ini, EXCLUSION constraint tetap menjaga integritas data, namun lebih fleksibel untuk menangani kebutuhan bisnis nyata.

### h. Rangkuman Sintaks EXCLUSION

Bagian ini merangkum pola umum saat menggunakan **EXCLUSION constraint** di PostgreSQL. Tujuannya adalah memberi gambaran jelas tentang struktur sintaks, komponen-komponen penting, serta operator yang paling sering digunakan. Dengan memahami template ini, Anda bisa menyusun constraint yang lebih kompleks sesuai kebutuhan aplikasi—misalnya untuk menghindari booking yang overlap, mencegah duplikasi, atau membuat aturan khusus berdasarkan kondisi tertentu.

#### Template Lengkap

Berikut bentuk umum sebuah EXCLUSION constraint dalam definisi tabel:

```sql
CREATE TABLE table_name (
    column1 TYPE,
    column2 TYPE,
    ...
    EXCLUDE USING index_method (
        column1 WITH operator1,
        column2 WITH operator2,
        ...
    )
    WHERE (predicate)  -- Optional
);
```

Template ini menunjukkan bahwa sebuah constraint dapat melibatkan beberapa kolom sekaligus, masing-masing dipasangkan dengan operator tertentu. PostgreSQL kemudian akan menggunakan kombinasi kolom-operator tersebut untuk mendeteksi konflik antar-row.

#### Penjelasan Komponen

##### 1. Index Method (`USING gist`)

Bagian ini menentukan jenis index yang digunakan untuk mendukung EXCLUSION constraint. Umumnya:

- **`gist`** adalah pilihan utama, terutama untuk tipe data yang tidak cocok dengan B-Tree, seperti _range types_, _geometric types_, dan tipe kompleks lainnya.
- EXCLUSION tidak bisa bekerja tanpa index method—karena database perlu struktur cepat untuk melakukan pengecekan konflik.

##### 2. Pasangan `column WITH operator`

Ini adalah inti dari EXCLUSION constraint. Setiap pasangan mendefinisikan aturan konflik. Beberapa contoh:

- `room_id WITH =` → konflik jika nilai `room_id` sama
- `reservation_period WITH &&` → konflik jika dua range overlapped
- Anda bisa menambahkan lebih banyak pasangan untuk membuat rule yang lebih spesifik.

PostgreSQL akan menolak row baru jika **semua pasangan dibandingkan menghasilkan konflik**.

##### 3. Predicate Opsional (`WHERE (predicate)`)

Bagian ini membuat constraint menjadi **partial**—hanya berlaku pada row yang memenuhi kondisi tertentu.

Contohnya:

```sql
WHERE (booking_status != 'canceled')
```

Dengan predicate ini, row yang statusnya `'canceled'` akan diabaikan dari pemeriksaan konflik. Fitur ini sangat berguna untuk skenario bisnis yang lebih fleksibel, misalnya:

- reservasi yang dibatalkan tidak perlu ikut dicek,
- hanya reservasi aktif yang tidak boleh bertabrakan,
- atau hanya tipe tertentu dari booking yang harus dicegah overlapping.

#### Operator yang Umum Digunakan

Beberapa operator berikut sering muncul dalam EXCLUSION constraint:

- **`=`**: mengecek equality.
  Untuk dapat digunakan dalam EXCLUSION dengan GiST, Anda perlu extension `btree_gist`.

- **`&&`**: mengecek overlap pada _range types_.
  Ini adalah operator paling populer untuk membuat rule anti-overlapping seperti pada sistem booking atau penjadwalan.

- **`@>`**: “contains”—misalnya untuk range yang mencakup range lain.

- **`<@`**: “contained by”—range yang berada di dalam range lain.

- Operator lainnya tergantung tipe data yang digunakan, karena PostgreSQL mendukung banyak jenis operator untuk GiST index.

Semua elemen di atas bekerja bersama untuk memberi Anda kontrol yang sangat fleksibel dalam mencegah data konflik—lebih kuat dan lebih ekspresif dibanding sekadar unique constraint.

## 3. Hubungan Antar Konsep

### Alur Pemahaman:

```
1. UNIQUE Constraint (dari video sebelumnya)
   ↓
   Terbatas pada equality check (nilai harus exact berbeda)

2. EXCLUSION Constraint
   ↓
   Lebih fleksibel: bisa gunakan operator custom (tidak hanya =)

3. Simple EXCLUSION
   ↓
   Cek satu kolom saja (terlalu ketat)

4. Composite EXCLUSION
   ↓
   Cek kombinasi kolom (lebih realistic)
   ↓
   Butuh btree_gist untuk mix equality + ranges

5. Partial EXCLUSION
   ↓
   Tambahkan WHERE clause untuk business logic kompleks
```

### Diagram Comparison:

```
UNIQUE Constraint:
├─ Operator: = (equality only)
├─ Use case: Prevent exact duplicates
└─ Example: email, username

EXCLUSION Constraint:
├─ Operator: Custom (=, &&, @>, dll.)
├─ Use case: Prevent complex conflicts
└─ Example: overlapping reservations, spatial conflicts
```

### Evolusi Solusi Reservasi:

```
Problem 1: Overlapping reservations
└─ Solusi: EXCLUDE (reservation_period WITH &&)
   └─ Problem baru: Tidak bisa book room berbeda di waktu sama

Problem 2: Multi-room support
└─ Solusi: EXCLUDE (room_id WITH =, reservation_period WITH &&)
   └─ Butuh: btree_gist extension
   └─ Problem baru: Canceled reservation tidak bisa di-book ulang

Problem 3: Canceled reservations
└─ Solusi: Tambahkan WHERE (booking_status != 'canceled')
   └─ Final solution! ✅
```

### Dependency Tree:

```
EXCLUSION Constraint
├─ Requires: GIST index knowledge
├─ Requires: Range types (TSRANGE, DATERANGE)
├─ Requires: Operator knowledge (&&, =, @>)
├─ Optional: btree_gist extension (untuk equality)
└─ Optional: WHERE clause (untuk partial constraint)
```

## 4. Kesimpulan

**Key Takeaways:**

1. **EXCLUSION vs UNIQUE:**

   - UNIQUE: Prevent exact duplicates (operator `=` only)
   - EXCLUSION: Prevent custom conflicts (any operator)

2. **Struktur dasar:**

   ```sql
   EXCLUDE USING gist (column WITH operator)
   ```

3. **Composite exclusion:**

   - Combine multiple columns dengan operators berbeda
   - Example: `room_id WITH =, reservation_period WITH &&`

4. **btree_gist extension:**

   - Diperlukan untuk menggunakan operator `=` dalam GIST index
   - Enable dengan: `CREATE EXTENSION btree_gist`

5. **Partial exclusion dengan WHERE:**

   - Tambahkan predicate untuk business logic
   - Example: `WHERE (booking_status != 'canceled')`

6. **Use cases umum:**
   - Overlapping time ranges (reservasi, jadwal)
   - Spatial data conflicts
   - Resource allocation
   - Appointment scheduling

**Best Practices:**

- Gunakan EXCLUSION ketika UNIQUE tidak cukup
- Pertimbangkan apakah butuh composite exclusion
- Install btree_gist jika mix equality dengan ranges
- Gunakan partial exclusion untuk business rules kompleks
- EXCLUSION lebih jarang digunakan dari UNIQUE, tapi sangat powerful untuk kasus tertentu

**Pattern umum:**

```sql
-- Pattern untuk reservasi/booking system
EXCLUDE USING gist (
    resource_id WITH =,      -- Resource yang sama
    time_range WITH &&       -- Tidak boleh overlap
)
WHERE (status != 'canceled') -- Kecuali yang canceled
```

**Catatan penting:**

- Foreign key constraints akan dibahas di modul indexing
- GIST dan index types akan dijelaskan lebih detail nanti
- Extensions akan dibahas di modul terpisah
