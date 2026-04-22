# 📝 Menggabungkan Multiple Indexes di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas kemampuan PostgreSQL untuk **menggabungkan dua index terpisah** saat menjalankan query dengan kondisi `AND` atau `OR`. Meskipun _composite index_ (index gabungan dalam satu struktur) umumnya lebih cepat untuk kondisi `AND`, PostgreSQL cukup cerdas untuk mengombinasikan dua _separate index_ menggunakan operasi **Bitmap AND / Bitmap OR** — yang masih jauh lebih baik daripada _full table scan_. Penting untuk memahami kapan harus menggunakan strategi mana, berdasarkan pola akses data (_access pattern_) dan bentuk query yang paling sering dijalankan.

---

## 2. Konsep Utama

### a. Composite Index vs. Two Separate Indexes

#### Apa itu Composite Index dan Two Separate Indexes?

**Composite index** adalah index yang dibuat dari **lebih dari satu kolom** dalam satu struktur (biasanya B-tree). Artinya, database menyimpan kombinasi nilai kolom tersebut dalam satu index yang terurut.

Sebaliknya, **two separate indexes** berarti kita membuat **dua index terpisah**, masing-masing hanya untuk satu kolom.

Contohnya:

- Composite index: `(first_name, last_name)`
- Two separate indexes: `(first_name)` dan `(last_name)` masing-masing berdiri sendiri

#### Bagaimana PostgreSQL Menggunakan Dua Index Terpisah?

Menariknya, PostgreSQL cukup pintar. Ketika kamu punya dua index terpisah, database tetap bisa **menggunakan keduanya dalam satu query**.

Caranya adalah dengan:

1. Melakukan scan pada masing-masing index
2. Menghasilkan hasil sementara dalam bentuk bitmap
3. Menggabungkan bitmap tersebut dengan operasi logika (biasanya `AND`)

Meskipun ini bukan cara paling optimal, hasilnya tetap jauh lebih baik dibanding:

- Melakukan **full table scan** (membaca seluruh tabel)
- Menggunakan satu index saja lalu melakukan filtering tambahan setelahnya

#### Ilustrasi Perbandingan Strategi

Misalkan ada query:

```sql
SELECT * FROM users
WHERE first_name = 'Aaron'
  AND last_name = 'Francis';
```

##### Strategi 1 — Two Separate Indexes (Bitmap AND)

```
[Scan index first_name] → hasil bitmap A
[Scan index last_name]  → hasil bitmap B
Bitmap AND(A, B)        → baris yang cocok di kedua index
→ Ambil baris dari heap
```

Penjelasan alurnya:

- PostgreSQL mencari semua baris dengan `first_name = 'Aaron'` → menghasilkan bitmap A
- Lalu mencari semua baris dengan `last_name = 'Francis'` → menghasilkan bitmap B
- Kedua bitmap ini digabung dengan operasi `AND`
- Hasil akhirnya adalah baris yang memenuhi kedua kondisi
- Setelah itu, baris diambil dari heap (data asli)

Pendekatan ini cukup efisien, tapi tetap ada overhead karena:

- Harus scan dua index
- Harus menggabungkan hasilnya

##### Strategi 2 — Composite Index (Lebih Cepat untuk AND)

```
[Scan index (first_name, last_name)] → langsung temukan baris
→ Ambil baris dari heap
```

Penjelasan:

- Karena kedua kolom sudah digabung dalam satu index, PostgreSQL bisa langsung mencari kombinasi `('Aaron', 'Francis')`
- Tidak perlu menggabungkan hasil dari dua index
- Prosesnya lebih langsung dan biasanya lebih cepat

#### Kesimpulan Intuitif

Kalau dianalogikan:

- **Two separate indexes** itu seperti mencari nama depan di satu buku, nama belakang di buku lain, lalu mencocokkan hasilnya
- **Composite index** itu seperti punya satu buku yang sudah berisi kombinasi nama depan + nama belakang — jadi pencariannya langsung

#### Kapan Harus Pakai yang Mana?

- Gunakan **composite index** jika:
  - Query sering menggunakan kondisi `AND` pada kolom yang sama
  - Urutan kolom dalam query konsisten

- Gunakan **separate indexes** jika:
  - Kolom sering digunakan secara independen dalam query berbeda
  - Kombinasi kolom tidak selalu digunakan bersama

Intinya, composite index unggul dalam performa untuk kondisi gabungan (`AND`), sementara separate indexes lebih fleksibel.

### b. Bitmap Index Scan: AND dan OR

#### Apa itu Bitmap Index Scan?

Ketika PostgreSQL ingin menggunakan **lebih dari satu index dalam satu query**, ia tidak langsung mengambil baris satu per satu. Sebaliknya, ia menggunakan pendekatan yang disebut **Bitmap Index Scan**.

Intinya, PostgreSQL:

1. Melakukan scan pada masing-masing index
2. Mengubah hasilnya menjadi **bitmap** (array berisi 0 dan 1)
3. Menggabungkan bitmap tersebut dengan operasi logika seperti `AND` atau `OR`
4. Baru kemudian mengambil baris yang sesuai dari heap

Pendekatan ini membuat proses penggabungan beberapa index menjadi jauh lebih efisien.

#### Bitmap AND (untuk kondisi AND)

Bitmap AND digunakan ketika query memiliki kondisi `AND`.

Alurnya:

- PostgreSQL scan index pertama (misalnya `first_name`)
- Lalu scan index kedua (misalnya `last_name`)
- Masing-masing menghasilkan bitmap
- Kedua bitmap digabung dengan operasi **AND**
- Hanya baris yang bernilai `1` di **keduanya** yang akan diambil

Contoh query:

```sql
SELECT * FROM users
WHERE first_name = 'Aaron' AND last_name = 'Francis';
```

Artinya, database hanya akan mengambil baris yang memenuhi **kedua kondisi sekaligus**.

#### Bitmap OR (untuk kondisi OR)

Bitmap OR digunakan ketika query memiliki kondisi `OR`.

Alurnya hampir sama:

- Scan kedua index
- Buat bitmap masing-masing
- Gabungkan dengan operasi **OR**
- Baris yang cocok di salah satu (atau keduanya) akan diambil

Contoh query:

```sql
SELECT * FROM users
WHERE first_name = 'Aaron' OR last_name = 'Francis';
```

Di sini, baris akan diambil jika memenuhi **minimal satu kondisi**.

#### Memahami dengan Analogi Bitmap

Agar lebih kebayang, anggap setiap index menghasilkan bitmap seperti ini:

```
Index first_name hasil:  [1, 0, 1, 0, 1]  (baris 1, 3, 5 cocok)
Index last_name  hasil:  [1, 1, 0, 0, 1]  (baris 1, 2, 5 cocok)
```

##### Hasil Bitmap AND

```
[1, 0, 0, 0, 1]
```

Penjelasan:

- Hanya posisi yang bernilai `1` di **kedua bitmap** yang dipertahankan
- Jadi hasilnya: baris ke-1 dan ke-5

##### Hasil Bitmap OR

```
[1, 1, 1, 0, 1]
```

Penjelasan:

- Semua posisi yang bernilai `1` di **salah satu bitmap** akan diambil
- Jadi hasilnya: baris ke-1, 2, 3, dan 5

#### Kenapa Pendekatan Ini Penting?

Tanpa bitmap:

- PostgreSQL harus mengambil baris satu per satu dari setiap index
- Lalu mencocokkannya secara manual (lebih mahal)

Dengan bitmap:

- Proses penggabungan dilakukan di level struktur data yang ringan
- Pengambilan baris dari heap hanya dilakukan sekali, setelah hasil final diketahui

#### Intuisi Sederhana

- **Bitmap AND** = irisan (intersection) → ambil yang memenuhi semua kondisi
- **Bitmap OR** = gabungan (union) → ambil yang memenuhi salah satu kondisi

Pendekatan ini membuat penggunaan multiple index tetap efisien, meskipun tidak secepat composite index untuk kasus `AND`.

### c. Perilaku PostgreSQL saat Composite Index Tersedia

#### Bagaimana PostgreSQL Memilih Index?

Ketika dalam sebuah tabel tersedia:

- dua index terpisah (`first_name` dan `last_name`)
- serta satu **composite index** (`first_name, last_name`)

PostgreSQL tidak langsung memilih salah satu secara kaku. Ia menggunakan komponen bernama **query planner**, yaitu sistem internal yang:

- menganalisis struktur index
- melihat statistik data (jumlah baris, distribusi nilai, dll)
- lalu memilih strategi yang paling efisien untuk query tertentu

Jadi, pilihan index itu **dinamis**, tergantung pada jenis query yang dijalankan.

#### Perilaku untuk Kondisi AND

Untuk query dengan kondisi `AND`, PostgreSQL **hampir selalu memilih composite index**.

Kenapa?

Karena:

- Composite index menyimpan kombinasi kolom dalam satu struktur B-tree
- PostgreSQL cukup melakukan **satu kali traversal index**
- Tidak perlu menggabungkan hasil dari beberapa index

Contohnya:

```sql id="h8o3y1"
SELECT * FROM users
WHERE first_name = 'Aaron'
  AND last_name = 'Francis';
```

Dengan adanya index:

```sql id="f5f0n2"
CREATE INDEX first_last ON users (first_name, last_name);
```

Maka PostgreSQL bisa langsung:

- mencari pasangan nilai `('Aaron', 'Francis')` di dalam satu index
- lalu mengambil baris dari heap

Hasilnya:

- lebih sedikit operasi
- lebih cepat dibanding Bitmap AND dari dua index terpisah

#### Perilaku untuk Kondisi OR

Untuk kondisi `OR`, situasinya berbeda.

Contoh:

```sql id="0zj5re"
SELECT * FROM users
WHERE first_name = 'Aaron'
   OR last_name = 'Francis';
```

Di sini, PostgreSQL **tidak selalu menggunakan composite index**, bahkan sering kali **tetap memilih dua index terpisah**.

Alasannya:

- Struktur B-tree pada composite index tidak dirancang untuk menangani pencarian `OR` secara efisien
- Composite index lebih optimal untuk pencarian yang mengikuti urutan kolom (terutama untuk `AND`)
- Untuk `OR`, lebih masuk akal:
  - scan masing-masing index secara terpisah
  - lalu gabungkan dengan **Bitmap OR**

Sehingga strateginya menjadi:

- Scan index `first_name`
- Scan index `last_name`
- Gabungkan hasilnya dengan OR
- Ambil baris dari heap

#### Contoh Setup Index

Misalkan kita punya index seperti ini:

```sql id="q8rjhz"
-- Index terpisah
CREATE INDEX first ON users (first_name);
CREATE INDEX last  ON users (last_name);

-- Composite index
CREATE INDEX first_last ON users (first_name, last_name);
```

Dengan kondisi ini:

- Query dengan `AND`
  → PostgreSQL cenderung memilih `first_last`
  → karena lebih langsung dan efisien

- Query dengan `OR`
  → PostgreSQL cenderung memilih `first` + `last`
  → lalu menggabungkannya dengan Bitmap OR

#### Intuisi Sederhana

- **Composite index** unggul untuk `AND` karena semua kondisi bisa dicek dalam satu jalur
- **Separate indexes + Bitmap OR** lebih cocok untuk `OR` karena hasilnya memang perlu digabungkan

#### Catatan Penting

Meskipun ini adalah perilaku umum, tetap ingat:

- PostgreSQL bisa saja memilih strategi berbeda jika statistik data berubah
- Query planner selalu mencoba mencari biaya eksekusi paling rendah, bukan mengikuti aturan kaku

Itulah kenapa memahami cara kerja ini penting — supaya kita bisa mendesain index yang benar-benar sesuai dengan pola query yang sering dijalankan.

### d. Kapan Menggunakan Strategi Mana?

#### Tidak Ada Jawaban Universal

Dalam praktik nyata, tidak ada satu strategi index yang selalu paling benar untuk semua kasus. Pilihan terbaik sangat bergantung pada **pola query (access pattern)** yang sering dijalankan oleh aplikasi.

Karena itu, penting untuk memahami karakteristik masing-masing pendekatan, lalu menyesuaikannya dengan kebutuhan.

#### Panduan Berdasarkan Jenis Kondisi

Mari kita uraikan berdasarkan tipe kondisi yang umum digunakan dalam query.

##### Kondisi AND (Semua Harus Terpenuhi)

Untuk kondisi `AND`, strategi terbaik biasanya adalah **composite index**.

Kenapa?

- PostgreSQL cukup melakukan satu kali traversal pada satu B-tree
- Semua kondisi dicek sekaligus dalam satu struktur
- Tidak perlu menggabungkan hasil dari beberapa index

Contoh:

```sql id="7s6q9c"
SELECT * FROM users
WHERE first_name = 'Aaron'
  AND last_name = 'Francis';
```

Dengan composite index `(first_name, last_name)`, query ini bisa dieksekusi dengan sangat efisien karena pencarian langsung mengarah ke kombinasi nilai yang spesifik.

##### Kondisi OR (Salah Satu Cukup)

Untuk kondisi `OR`, biasanya lebih efektif menggunakan:
**dua index terpisah + Bitmap OR**

Kenapa bukan composite index?

- Struktur B-tree tidak dirancang optimal untuk pencarian OR lintas kolom
- Lebih efisien scan masing-masing index lalu menggabungkan hasilnya

Contoh:

```sql id="o2dklv"
SELECT * FROM users
WHERE first_name = 'Aaron'
   OR last_name = 'Francis';
```

Di sini PostgreSQL akan:

- Scan index `first_name`
- Scan index `last_name`
- Menggabungkan hasilnya dengan operasi OR (bitmap)

Pendekatan ini lebih fleksibel untuk kondisi seperti ini.

##### Tidak Ada Index Sama Sekali

Jika tidak ada index yang relevan, PostgreSQL tidak punya pilihan selain melakukan:

**Full table scan**

Artinya:

- Seluruh tabel dibaca dari awal sampai akhir
- Setiap baris diperiksa satu per satu

Ini adalah skenario paling lambat, terutama jika tabel berukuran besar. Karena itu, sebisa mungkin **hindari kondisi ini pada query yang sering dijalankan**.

#### Cara Menentukan Strategi yang Tepat

Daripada menebak-nebak, pendekatan yang lebih tepat adalah berbasis data dan observasi.

Berikut panduan praktisnya:

- Identifikasi query yang paling sering digunakan (hot paths di aplikasi)
- Perhatikan apakah lebih sering menggunakan `AND` atau `OR`
- Buat index yang sesuai dengan pola tersebut
- Uji performanya secara langsung

#### Gunakan EXPLAIN ANALYZE untuk Validasi

PostgreSQL menyediakan tool penting untuk melihat bagaimana query dieksekusi, yaitu `EXPLAIN ANALYZE`.

Contoh penggunaan:

```sql id="95x3g5"
EXPLAIN ANALYZE
SELECT * FROM users
WHERE first_name = 'Aaron' AND last_name = 'Francis';
```

Dari hasil ini, kamu bisa melihat:

- Apakah PostgreSQL menggunakan index atau tidak
- Jenis scan yang dipakai (Index Scan, Bitmap Index Scan, dll)
- Estimasi vs real execution time

Ini penting karena:

- Query planner bekerja berdasarkan estimasi
- Data nyata di database bisa menghasilkan keputusan berbeda dari teori

#### Intuisi Akhir

- Gunakan **composite index** untuk kondisi yang selalu dipakai bersama (`AND`)
- Gunakan **separate indexes** untuk kondisi yang fleksibel atau sering dipakai sendiri-sendiri
- Selalu validasi dengan data nyata, bukan asumsi

Dengan pendekatan ini, kamu tidak hanya memahami teori indexing, tapi juga bisa menerapkannya secara tepat di dunia nyata.

## 3. Hubungan Antar Konsep

Semua konsep di atas membentuk satu alur pemahaman yang saling terhubung:

1. **B-tree** adalah struktur dasar di balik setiap index PostgreSQL. Satu B-tree dioptimalkan untuk satu kondisi pencarian; semakin spesifik, semakin cepat traversal-nya.

2. **Bitmap index scan** adalah mekanisme yang memungkinkan PostgreSQL "menjembatani" dua index terpisah — ia mengonversi hasil scan menjadi representasi bitmap, lalu menggabungkannya dengan operasi AND atau OR sesuai kondisi query.

3. **Composite index** memanfaatkan satu B-tree untuk dua (atau lebih) kolom sekaligus. Ini sangat efisien untuk `AND` karena seluruh kondisi dapat dievaluasi dalam satu traversal. Namun untuk `OR`, keunggulan ini hilang — justru dua B-tree terpisah yang di-bitmap-OR lebih cocok.

4. **Query planner PostgreSQL** secara otomatis memilih strategi terbaik berdasarkan statistik tabel dan kondisi query — sehingga kita sebagai developer tidak harus memaksakan satu strategi, melainkan menyediakan index yang tepat dan membiarkan PostgreSQL memutuskan.

Alur keputusan yang utuh:

```
Query diterima
     ↓
Ada composite index yang sesuai?
  → Ya, kondisi AND → gunakan composite index (satu B-tree)
  → Ya, kondisi OR  → mungkin tetap pakai dua index terpisah + Bitmap OR
  → Tidak           → gunakan dua index terpisah jika ada, atau table scan
```

---

## 4. Kesimpulan

- PostgreSQL mampu menggabungkan **dua index terpisah** menggunakan **Bitmap AND** (untuk `AND`) atau **Bitmap OR** (untuk `OR`), tanpa perlu composite index.
- Menggabungkan dua index terpisah **lebih lambat** dari composite index untuk kondisi `AND`, tetapi **lebih cepat** dari _table scan_ atau filtering manual dari heap.
- Untuk kondisi **`OR`**, dua index terpisah yang dikombinasikan justru bisa **lebih baik** dari composite index — karena struktur B-tree kurang cocok untuk OR.
- Strategi terbaik selalu bergantung pada **pola akses query** Anda. Gunakan `EXPLAIN ANALYZE`, uji pada data nyata, dan bangun strategi index berdasarkan hasil pengukuran — bukan asumsi.
- PostgreSQL yang secara otomatis memilih strategi index ini adalah fitur yang sangat berguna (_"Postgres has your back"_), namun tetap perlu dipahami dan diuji oleh developer.
