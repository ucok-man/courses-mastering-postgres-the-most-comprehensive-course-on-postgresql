# 📝 Range Types di PostgreSQL: Menyimpan dan Query Data Rentang

## 1. Ringkasan Singkat

Video ini membahas **range types** di PostgreSQL, yaitu tipe data yang merepresentasikan rentang nilai dengan **lower bound** dan **upper bound**. Range adalah meta type yang bisa memiliki berbagai subtype (integer, numeric, date, timestamp). Pembahasan mencakup cara membuat range, perbedaan antara **inclusive/exclusive bounds**, **continuous vs discrete ranges**, operasi query (containment, overlap, intersection), serta pengenalan **multi-range** untuk rentang non-contiguous.

---

## 2. Konsep Utama

### a. Apa itu Range Types?

Range types di PostgreSQL adalah **meta type** yang digunakan untuk merepresentasikan sebuah rentang nilai—selalu terdiri dari batas bawah (lower bound) dan batas atas (upper bound). Yang dimaksud meta type di sini adalah bahwa range bukan tipe data berdiri sendiri, tetapi selalu terhubung dengan sebuah **subtype**. Dengan kata lain, range hanyalah “wadah” yang memegang rentang dari suatu tipe data tertentu.

#### Karakteristik Range Types

Range memiliki beberapa sifat penting yang membuatnya fleksibel dan sangat berguna, terutama untuk data yang berhubungan dengan periode waktu, interval angka, atau batasan tertentu.

1. **Bukan tipe data standalone**
   Range harus memiliki subtype. Misalnya, ada range untuk `integer`, `numeric`, atau `date`. Kita tidak bisa membuat range tanpa menentukan tipe data apa yang direntanginya.

2. **Memiliki subtype spesifik**
   Subtype menentukan jenis nilai yang berada di dalam rentang tersebut.
   Contohnya:

   - `INT4RANGE` → range untuk integer 32-bit
   - `DATERANGE` → range untuk nilai tanggal

3. **Bounds (batas) bisa inclusive atau exclusive**
   Ini menunjukkan apakah nilai batas ikut dihitung atau tidak.

   - Inclusive menggunakan tanda: `[` atau `]`
   - Exclusive menggunakan tanda: `(` atau `)`

   Misalnya:

   ```sql
   SELECT '[1,5]'::int4range;  -- 1 sampai 5, kedua nilai termasuk
   SELECT '(1,5)'::int4range;  -- 2 sampai 4 (1 dan 5 tidak termasuk)
   ```

4. **Bisa unbounded (tanpa batas)**
   Kadang kita hanya peduli pada batas bawah atau batas atas.
   PostgreSQL mengizinkan range tanpa batas, misalnya:

   ```sql
   SELECT '[10,)'::int4range;  -- mulai dari 10 ke atas tanpa batas
   ```

#### Jenis-jenis Range Types

PostgreSQL menyediakan beberapa tipe range bawaan yang umum dipakai:

- **`INT4RANGE`**
  Untuk rentang integer 4-byte (standard int).

- **`INT8RANGE`**
  Untuk rentang integer 8-byte (`bigint`).

- **`NUMRANGE`**
  Untuk rentang nilai numerik bertipe `numeric`. Cocok untuk data continuous seperti harga atau persentase.

- **`DATERANGE`**
  Untuk rentang tanggal. Tipe ini bersifat discrete, artinya PostgreSQL mengerti bahwa ada “loncatan 1 hari”.

- **`TSRANGE`**
  Untuk rentang `timestamp without time zone`.

- **`TSTZRANGE`**
  Untuk rentang `timestamp with time zone`.

#### Contoh Penggunaan Sederhana

Misalnya kita ingin menyimpan periode sewa ruang:

```sql
CREATE TABLE bookings (
  id SERIAL PRIMARY KEY,
  period TSRANGE NOT NULL
);
```

Insert data:

```sql
INSERT INTO bookings (period)
VALUES ('[2025-01-01 08:00, 2025-01-01 12:00)');
```

Lalu, kita bisa mengecek apakah sebuah waktu tertentu berada di dalam rentang:

```sql
SELECT
  period @> '2025-01-01 09:30'::timestamp AS is_within_range
FROM bookings;
```

Hasilnya:

```
 is_within_range
----------------
 t
```

Artinya waktu tersebut berada dalam rentang booking.

Range types memudahkan kita bekerja dengan interval nilai secara aman, konsisten, dan efisien tanpa perlu menghitung manual apakah suatu angka atau tanggal berada dalam sebuah periode tertentu.

### b. Cara Membuat Range

Untuk membuat range di PostgreSQL, ada dua pendekatan utama: menggunakan string literal yang di-cast ke tipe range, atau menggunakan constructor function bawaan. Keduanya menghasilkan objek range yang sama, tetapi cara penulisannya sedikit berbeda. Berikut penjelasan yang lebih runtut dan mudah dipahami.

#### Metode 1: Menggunakan String Literal dengan Casting

Pada metode ini, kita menulis representasi range dalam bentuk string, lalu melakukan casting ke tipe range yang kita inginkan. Format dasar string-nya adalah:

```
'[lower,upper]'::rangetype
```

Contoh:

```sql
SELECT '[1,5]'::INT4RANGE;
```

Menariknya, PostgreSQL akan menyesuaikan format output sesuai aturan subtype-nya. Untuk `INT4RANGE`, batas atas dianggap **exclusive** secara default. Maka meskipun kita menulis `[1,5]`, PostgreSQL akan menampilkan:

```
[1,6)
```

Artinya, range tersebut mencakup angka 1 sampai sebelum 6. PostgreSQL melakukan penyesuaian ini karena range untuk integer bersifat _discrete_, sehingga `[1,5]` dipahami sebagai semua nilai 1 sampai 5 inclusively, lalu dikonversi ke bentuk standar `[1,6)`.

Contoh lain:

```sql
SELECT '[1,5]'::NUMRANGE;
```

Untuk tipe `NUMRANGE` (yang bersifat _continuous_), format output akan tetap mengikuti input, sehingga hasilnya:

```
[1,5]
```

Karena tidak perlu ada penyesuaian seperti pada integer.

#### Metode 2: Menggunakan Constructor Functions

Metode kedua menggunakan fungsi pembentuk range. Format dasarnya:

```
rangetype(lower, upper)
```

Secara default, constructor akan membuat **lower inclusive** dan **upper exclusive**, yang ditandai sebagai `[lower,upper)`.

Contoh:

```sql
SELECT NUMRANGE(1, 5);
```

Output:

```
[1,5)
```

Hal yang sama berlaku untuk range integer:

```sql
SELECT INT4RANGE(1, 5);
```

Output:

```
[1,5)
```

Namun, jika kita ingin mengontrol bounds secara lebih spesifik, kita bisa menambahkan parameter ketiga berupa string:

- `'[]'` → kedua inclusive
- `'()'` → kedua exclusive
- `'(]'` → bawah exclusive, atas inclusive
- `'[)'` → bawah inclusive, atas exclusive (default)

Contoh lengkapnya:

```sql
SELECT NUMRANGE(1, 5, '[]');   -- kedua inclusive
-- Output: [1,5]

SELECT NUMRANGE(1, 5, '()');   -- kedua exclusive
-- Output: (1,5)

SELECT NUMRANGE(1, 5, '(]');   -- lower exclusive, upper inclusive
-- Output: (1,5]

SELECT NUMRANGE(1, 5, '[)');   -- default
-- Output: [1,5)
```

#### Notasi Bounds

Untuk memahami range lebih jelas, ingat bahwa simbol bounds memiliki makna khusus:

- `[` atau `]` → **inclusive**, nilai tersebut ikut menjadi bagian dari range
- `(` atau `)` → **exclusive**, nilai tersebut tidak termasuk

Contoh sederhana:

- `[1,5]` berarti 1 sampai 5, keduanya termasuk
- `(1,5)` berarti lebih besar dari 1 sampai kurang dari 5
- `[1,5)` berarti 1 termasuk, 5 tidak termasuk

Dengan memahami dua metode pembuatan range dan cara kerja bound-nya, kita bisa membangun range sesuai kebutuhan dengan cara yang paling nyaman atau paling sesuai dengan konteks query kita.

### c. Discrete vs Continuous Ranges

Dalam PostgreSQL, range type dibagi menjadi dua kategori besar: **discrete** dan **continuous**. Memahami perbedaannya sangat penting karena PostgreSQL memperlakukan kedua jenis range ini secara berbeda, terutama ketika menyangkut konversi bounds dan interpretasi nilai di dalam rentang tersebut.

#### Discrete Range

Discrete range adalah range yang memiliki **step** atau satuan yang jelas dan terhitung. Subtype ini memiliki nilai yang dapat “dilompati” satu per satu secara pasti, misalnya 1 → 2 → 3 atau hari ke hari.

Beberapa contoh tipe range yang termasuk kategori discrete:

- `INT4RANGE` (integer 4-byte)
- `INT8RANGE` (bigint)
- `DATERANGE` (tanggal harian)

Karena nilainya diskrit, PostgreSQL sering melakukan **normalisasi** pada bounds. Artinya, ia mengubah representasi range Anda ke bentuk standar, tapi tanpa mengubah makna rentangnya.

Contoh:

```sql
SELECT '[1,5]'::INT4RANGE;
```

Output:

```
[1,6)
```

Pada integer, `[1,5]` dan `[1,6)` mewakili nilai yang sama: 1, 2, 3, 4, dan 5. PostgreSQL memilih bentuk `[1,6)` sebagai format standard karena bagian atas untuk range integer biasanya dianggap exclusive dengan step sebesar 1.

Jadi ketika Anda memasukkan `[1,5]`, PostgreSQL memahaminya sebagai “mulai dari 1 sampai 5 termasuk,” lalu mengubahnya ke “[1 sampai sebelum 6]” supaya konsisten.

#### Continuous Range

Berbeda dengan discrete, continuous range tidak memiliki step yang terhitung secara pasti. Di antara dua nilai selalu ada nilai lain—bahkan tak terbatas. Karena sifatnya ini, PostgreSQL **tidak** melakukan normalisasi bounds pada continuous range.

Contoh tipe continuous:

- `NUMRANGE`
- `TSRANGE`
- `TSTZRANGE`

Karena setiap titik dalam rentang bisa memiliki nilai antara yang tak terhingga, `[1,5]` tidak sama dengan `[1,6)`.

Contoh:

```sql
SELECT '[1,5]'::NUMRANGE;
```

Output:

```
[1,5]
```

PostgreSQL tidak mengubahnya menjadi `[1,6)` karena nilai seperti `5.1`, `5.01`, `5.001`, dan seterusnya adalah nilai valid yang berada di antara 5 dan 6. Jika range itu diubah otomatis, maka maknanya berubah total—dan itu tidak diinginkan pada continuous range.

Perbandingan lain:

```sql
SELECT '[1,6)'::NUMRANGE;
```

Output:

```
[1,6)
```

Range ini mencakup semua nilai dari 1 sampai sebelum 6, termasuk 5.99, 5.999, dan seterusnya. Nilai tersebut **tidak valid** di `[1,5]`, jadi keduanya jelas berbeda.

#### Diagram Perbandingan

```
Discrete Range (INT4RANGE):
[1,5]  ≡  [1,6)
└─ nilai valid: 1, 2, 3, 4, 5

Continuous Range (NUMRANGE):
[1,5]  ≠  [1,6)
└─ [1,5]:  valid sampai tepat 5.0
└─ [1,6): valid sampai 5.999... (sebelum 6)
```

Dengan memahami perbedaan discrete vs continuous, Anda bisa menghindari kebingungan ketika melihat PostgreSQL “mengubah” range tertentu atau ketika ingin bekerja presisi pada data bersifat waktu dan angka pecahan.

### d. Unbounded dan Empty Ranges

Range di PostgreSQL tidak selalu harus memiliki batas bawah atau batas atas. Kadang kita hanya ingin menunjukkan “mulai dari sini dan seterusnya,” atau sebaliknya. Di sisi lain, PostgreSQL juga memiliki konsep _empty range_, yaitu range yang sebenarnya tidak mewakili nilai apa pun. Keduanya sering membingungkan, jadi penting untuk memahami bagaimana keduanya bekerja.

#### Unbounded Range

Unbounded range adalah range yang tidak memiliki batas pada salah satu sisi, atau bahkan kedua sisi. Notasinya menggunakan tanda kosong `(` atau `)` tanpa nilai di sisi yang tidak dibatasi.

Contoh-contohnya:

```sql
SELECT '[5,)'::INT4RANGE;
```

Output:

```
[5,)
```

Artinya: rentang dimulai dari 5 dan berlangsung tanpa batas atas (hingga _infinity_). Ini berguna ketika kita ingin menyatakan bahwa sesuatu berlaku mulai angka tertentu tanpa akhir yang pasti.

Contoh lain, unbounded di bagian bawah:

```sql
SELECT '(,10]'::INT4RANGE;
```

Output:

```
(,10]
```

Artinya: rentang mencakup semua nilai dari minus infinity hingga 10 (termasuk 10).

Dan jika kita ingin menunjukkan seluruh nilai yang mungkin:

```sql
SELECT '(,)'::DATERANGE;
```

Output:

```
(,)
```

Range seperti ini mencakup semua tanggal yang mungkin ada.

#### Use Case Unbounded Range

Beberapa situasi nyata yang sangat cocok menggunakan unbounded:

- **Status hotel room "under construction"** yang belum diketahui kapan selesai. Kita bisa simpan rentang waktu yang dimulai dari tanggal tertentu tetapi tanpa batas akhir.
- **Produk "discontinued"** mulai tanggal tertentu, tetapi tidak ada tanggal berakhir karena status tersebut permanen.

Dengan unbounded range, kita tidak perlu mengisi nilai akhir sembarangan hanya untuk menjaga struktur data.

#### Empty Range

Berbeda dari unbounded, _empty range_ adalah range yang **tidak memiliki nilai sama sekali**. PostgreSQL memberikan kata kunci khusus `empty` untuk merepresentasikan kondisi ini.

Contoh:

```sql
SELECT 'empty'::INT4RANGE;
```

Output:

```
empty
```

Range ini bukan berarti “tidak berbatas,” melainkan benar-benar tidak berisi nilai apa pun. Misalnya, ini bisa terjadi saat hasil operasi intersect menghasilkan rentang yang tidak memiliki irisan.

#### Perbedaan Empty vs Unbounded

Perlu digarisbawahi bahwa keduanya adalah dua konsep yang sangat berbeda:

- `empty` berarti **tidak ada nilai yang valid dalam rentang tersebut**, seperti sebuah set kosong.
- `(,)` berarti **semua nilai mungkin berada dalam rentang**, dari minus infinity hingga infinity.

Dengan kata lain:

- `empty` = rentangnya **kosong**
- Unbounded `(,)` = rentangnya **seluruh kemungkinan**

Keduanya terlihat mirip secara visual, tetapi maknanya berlawanan. Memahami perbedaan ini sangat penting untuk menghindari bug ketika bekerja dengan operasi range seperti intersect, union, atau pengecekan containment.

### e. Operator Containment (`@>`)

Operator `@>` adalah salah satu fitur penting dalam range types di PostgreSQL. Operator ini digunakan untuk mengecek apakah sebuah **range mengandung nilai tertentu**. Jika nilai tersebut berada di dalam rentang (memperhatikan batas bawah/atas serta inclusive/exclusive), maka hasilnya adalah `true`. Kalau tidak, maka hasilnya `false`.

#### Sintaks Dasar

Bentuk penulisannya sangat sederhana:

```sql
range @> value
```

Artinya: “Apakah `range` ini **mengandung** `value`?”

#### Contoh Penggunaan Dasar

Mari kita mulai dengan membuat sebuah tabel dan mengisinya dengan beberapa range untuk diuji.

```sql
CREATE TABLE range_example (
  id SERIAL,
  int_range INT4RANGE
);

INSERT INTO range_example (int_range) VALUES
  ('[1,11)'::INT4RANGE),
  ('[2,101)'::INT4RANGE),
  ('[5,)'::INT4RANGE);
```

Tabel ini memiliki tiga baris dengan range yang berbeda-beda.
Sekarang, kita ingin mencari baris mana yang rentangnya mengandung angka 5:

```sql
SELECT * FROM range_example
WHERE int_range @> 5;
```

Output:

- Ketiga baris cocok, karena angka 5 berada di semua range tersebut:

  - `[1,11)` → 5 ada di antara 1 dan sebelum 11
  - `[2,101)` → 5 jelas ada
  - `[5,)` → batas bawahnya inclusive, jadi 5 termasuk

Berikutnya, mari coba nilai 11:

```sql
SELECT * FROM range_example
WHERE int_range @> 11;
```

Output:

- Hanya baris kedua (`[2,101)`) dan ketiga (`[5,)`) yang mengandung nilai 11.
- Baris pertama **tidak** cocok karena `[1,11)` adalah **upper exclusive**, artinya 11 tidak termasuk ke dalam range tersebut.

Contoh ini memperlihatkan bagaimana detail kecil seperti inclusive dan exclusive bisa mengubah hasilnya.

#### Pengaruh Inclusive dan Exclusive

Perbedaan antara inclusive dan exclusive sangat penting karena menentukan apakah sebuah nilai dianggap berada di dalam range atau tidak.

Contoh:

```sql
SELECT '[5,)'::INT4RANGE @> 5;
```

Output:

```
true
```

Karena `[5,)` memiliki batas bawah **inclusive**, maka nilai 5 termasuk dalam rentang.

Namun:

```sql
SELECT '(5,)'::INT4RANGE @> 5;
```

Output:

```
false
```

Kali ini batas bawahnya **exclusive**, sehingga range tersebut dimulai dari nilai yang lebih besar dari 5. Maka 5 **tidak** termasuk.

Memahami hal ini sangat penting terutama ketika bekerja dengan data yang sensitif terhadap batas, seperti jadwal waktu, interval harga, umur, dan sebagainya. Operator `@>` membantu Anda membuat query lebih ekspresif dan akurat dalam mengevaluasi apakah sebuah nilai berada dalam rentang tertentu.

### f. Operator Overlap (`&&`)

Operator `&&` digunakan ketika kita ingin mengecek apakah **dua range saling tumpang tindih**. Ini adalah operator yang sangat penting ketika bekerja dengan data berbasis rentang waktu, harga, interval angka, atau jadwal — intinya, ketika kita perlu tahu apakah dua interval memiliki bagian yang bersinggungan.

Pertanyaannya sederhana:
**Apakah range A dan range B memiliki setidaknya satu nilai yang sama-sama berada di dalam keduanya?**
Jika iya, maka hasilnya `true`. Kalau tidak, `false`.

#### Sintaks Dasar

```sql
range1 && range2
```

Anda bisa meletakkan literal range, kolom tipe range, atau kombinasi keduanya di kiri dan kanan operator.

#### Contoh Penggunaan

Berikut contoh query yang mengecek apakah nilai `int_range` pada tabel tumpang tindih dengan range `[10,20)`:

```sql
SELECT id, int_range
FROM range_example
WHERE int_range && '[10,20)'::INT4RANGE;
```

Hasilnya:

| id  | int_range | Alasan overlap                              |
| --- | --------- | ------------------------------------------- |
| 1   | [1,11)    | overlap di nilai 10                         |
| 2   | [2,101)   | overlap di 10–19                            |
| 3   | [5,)      | overlap di 10–19 (karena unbounded ke atas) |

Dari hasil tersebut, terlihat bahwa range `[10,20)` memiliki nilai-nilai yang berada di dalam ketiga range pada tabel.
Misalnya, `10` berada dalam `[1,11)`, dan angka apa pun antara 10–19 berada dalam range lain.

#### Contoh Edge Case: Batas Inclusive vs Exclusive

Kasus batas atas dan bawah sering kali menghasilkan hasil yang mengejutkan jika tidak diperhatikan.

Contoh:

```sql
SELECT '[1,11)'::INT4RANGE && '[11,20)'::INT4RANGE;
```

Output:

```
false
```

Alasannya:

- Range pertama adalah `[1,11)` → mencakup 1 hingga 10
- Range kedua adalah `[11,20)` → mencakup 11 hingga 19
- Tidak ada nilai yang berada di _kedua_ range
- Angka 11 _tidak termasuk_ dalam range pertama, sehingga tidak ada titik pertemuan

Versi lain:

```sql
SELECT '[1,11]'::INT4RANGE && '[11,20)'::INT4RANGE;
```

Output:

```
true
```

Karena:

- Range pertama `[1,11]` mencakup nilai 11
- Range kedua `[11,20)` juga mencakup nilai 11
- Mereka berbagi satu nilai yang sama, yaitu 11
- Maka keduanya **overlap**

Contoh ini menunjukkan bahwa detail kecil seperti penggunaan tanda `]` atau `)` bisa memengaruhi apakah dua range dianggap tumpang tindih. Karena itu, ketika menggunakan operator `&&`, penting untuk memastikan batas rangemu sudah ditentukan dengan benar sesuai kebutuhan aplikasi.

### g. Operator Intersection (`*`)

#### Penjelasan Umum

Operator **intersection (`*`)** digunakan untuk mencari **irisan** dari dua buah range, yaitu bagian nilai yang **muncul di kedua range secara bersamaan**. Jika dua range tidak memiliki area yang tumpang tindih, hasilnya akan berupa range kosong.

Sintaks penggunaannya sangat sederhana:

```sql
range1 * range2
```

Artinya, PostgreSQL akan menghitung area yang overlap antara `range1` dan `range2`.

---

#### Contoh Dasar

Mari kita lihat contoh pertama untuk memahami cara kerja operator ini:

```sql
-- Irisan [10,20) dan [15,25)
SELECT INT4RANGE(10, 20) * INT4RANGE(15, 25);
-- Output: [15,20)
```

Range pertama adalah **[10,20)**, artinya dimulai dari 10 (inklusif) hingga 20 (eksklusif).
Range kedua adalah **[15,25)**.

Bagian yang muncul di kedua range hanya berada pada nilai **15 sampai 19**, sehingga hasilnya adalah **[15,20)**.
PostgreSQL otomatis menghasilkan range irisan tersebut.

---

#### Contoh Dengan Inclusive Bound

Bound juga mempengaruhi hasil irisan. Contoh berikut menunjukkan bagaimana hasil bisa berbeda jika batas-batasnya inklusif:

```sql
SELECT INT4RANGE(10, 20, '[]') * INT4RANGE(15, 25);
-- Output: [15,21)
```

Range pertama sekarang adalah **[10,20]**, yaitu 10 hingga 20 dan **20-nya ikut dihitung**.
Untuk tipe range **discrete** seperti `int4range`, ada aturan khusus:

- `[15,20]` secara internal ekuivalen dengan `[15,21)` karena batas atasnya bersifat eksklusif dan akan dinaikkan satu tingkat.

Itulah sebabnya PostgreSQL menghasilkan **[15,21)**—yang sebenarnya mewakili nilai yang sama: 15 hingga 20.

---

#### Diagram Intersection

Ilustrasi visual di bawah ini menggambarkan bagaimana kedua range saling overlap:

```
Range 1:  [====10---15---20========)
Range 2:         [===15---20---25==)
                      ↓
Intersection:    [===15---20===)
```

Dari diagram tersebut terlihat jelas bahwa bagian yang beririsan dimulai dari 15 hingga 20. Operator `*` otomatis mengambil area overlap ini dan mengubahnya menjadi sebuah range baru.

### h. Fungsi Upper dan Lower

#### Penjelasan Umum

PostgreSQL menyediakan beberapa fungsi untuk membaca informasi penting dari sebuah range, seperti batas bawah, batas atas, dan apakah batas tersebut bersifat inklusif atau eksklusif. Fungsi-fungsi ini sangat membantu ketika kita ingin memproses atau memeriksa range secara dinamis, tanpa harus mengurai format teksnya secara manual.

Berikut adalah fungsi utamanya:

- `LOWER(range)` → mengambil **lower bound** (nilai batas bawah).
- `UPPER(range)` → mengambil **upper bound** (nilai batas atas) dalam bentuk **eksklusif**.
- `LOWER_INC(range)` → mengecek apakah lower bound bersifat **inclusive**.
- `UPPER_INC(range)` → mengecek apakah upper bound bersifat **inclusive**.

---

#### Contoh dengan Discrete Range

Mari kita lihat bagaimana fungsi-fungsi ini bekerja pada tipe range **discrete**, seperti `INT4RANGE`:

```sql
SELECT
  UPPER('[10,20]'::INT4RANGE) AS upper_val,
  UPPER_INC('[10,20]'::INT4RANGE) AS upper_inclusive;
-- Output: upper_val=21, upper_inclusive=false
```

Hasil di atas mungkin terlihat sedikit membingungkan: mengapa **upper_val = 21**, padahal range-nya `[10,20]` yang terlihat seperti berakhir di 20?

Penjelasannya:

- Range `[10,20]` berarti **10 sampai 20, dan 20 termasuk**.
- Untuk range discrete seperti integer, PostgreSQL akan merepresentasikan `[10,20]` secara internal sebagai **[10,21)**.
  Ini karena batas atas **selalu eksklusif** secara internal.
- `UPPER()` mengembalikan batas atas dalam bentuk eksklusif tersebut.
  Maka upper = 21.
- Karena batas atasnya eksklusif, `UPPER_INC()` bernilai `false`.

Dengan kata lain, walaupun kamu menuliskan 20 sebagai batas akhir, PostgreSQL menafsirkan nilai itu sebagai “hingga 20 termasuk,” lalu mengonversinya menjadi `[10,21)` untuk konsistensi.

---

#### Contoh dengan Continuous Range

Untuk range continuous seperti `NUMRANGE`, aturan internalnya berbeda. Perhatikan contoh berikut:

```sql
SELECT
  UPPER('[10,20)'::NUMRANGE) AS upper_val,
  UPPER_INC('[10,20)'::NUMRANGE) AS upper_inclusive;
-- Output: upper_val=20, upper_inclusive=false
```

Pada kasus ini, nilai upper tetap **20**, sesuai dengan yang kita tulis.
Mengapa tidak ditambah 1 seperti pada `INT4RANGE`?

Jawabannya karena **range continuous tidak memiliki step diskrit**. Tidak ada nilai “selanjutnya” dari 19.999…, sehingga tidak mungkin PostgreSQL merepresentasikannya sebagai `[10,20]` atau `[10,20.0...]`.

Karena itu, satu-satunya cara menyatakan batas atas eksklusif adalah: **“20, tapi 20 tidak termasuk.”**

---

#### Contoh Lengkap

Berikut contoh lengkap penggunaan keempat fungsi pada range integer:

```sql
SELECT
  LOWER('[10,20]'::INT4RANGE) AS lower_val,
  LOWER_INC('[10,20]'::INT4RANGE) AS lower_inclusive,
  UPPER('[10,20]'::INT4RANGE) AS upper_val,
  UPPER_INC('[10,20]'::INT4RANGE) AS upper_inclusive;
-- Output:
-- lower_val=10, lower_inclusive=true
-- upper_val=21, upper_inclusive=false
```

Dari hasil tersebut, kita bisa membaca secara jelas:

- Range dimulai dari **10**, dan batas bawahnya **inklusif**.
- Secara internal range berakhir di **21** (yang merepresentasikan 20 inklusif).
- Batas atasnya bersifat **eksklusif**.

Dengan memahami fungsi-fungsi ini, kita bisa memanipulasi dan memeriksa range PostgreSQL dengan lebih percaya diri, tanpa harus menebak cara PostgreSQL menyimpan batas-batasnya di belakang layar.

### i. Multi-Range

#### Apa itu Multi-Range?

Multi-range adalah tipe data di PostgreSQL yang memungkinkan kita menyimpan **beberapa range sekaligus** dalam satu kolom. Bedanya dengan range biasa adalah multi-range dapat memuat rentang yang **tidak saling bersebelahan**. Dengan kata lain, kamu bisa menyimpan beberapa interval terpisah dan menganggapnya sebagai satu kesatuan.

PostgreSQL menyediakan berbagai tipe multi-range sesuai tipe data yang digunakan, misalnya untuk integer, bigint, number, date, dan timestamp.

Jenis-jenis multi-range yang umum digunakan:

- `INT4MULTIRANGE`
- `INT8MULTIRANGE`
- `NUMMULTIRANGE`
- `DATEMULTIRANGE`
- `TSMULTIRANGE`
- `TSTZMULTIRANGE`

Semua tipe ini bekerja dengan prinsip yang sama, hanya berbeda pada tipe datanya.

#### Contoh Dasar Multi-Range

Berikut contoh sederhana bagaimana PostgreSQL mewakili dua range angka yang terpisah:

```sql
-- Multi-range: 3-6 dan 8
SELECT '{[3,7), [8,9)}'::INT4MULTIRANGE;
-- Output: {[3,7),[8,9)}
-- Merepresentasikan nilai: 3,4,5,6,8
```

Di dalam contoh tersebut:

- Range pertama adalah `[3,7)` → mencakup 3 hingga 6.
  Nilai 7 tidak termasuk karena upper bound-nya eksklusif.
- Range kedua adalah `[8,9)` → hanya mencakup nilai 8.

Dengan dua range ini digabungkan, multi-range tersebut secara efektif menyatakan deretan nilai: **3, 4, 5, 6, dan 8**, dengan adanya gap di antara 6 dan 8.

---

#### Penggunaan Operator Containment

Operator containment (`@>`) tetap berlaku pada multi-range, sama seperti pada range biasa. Operator ini mengecek apakah sebuah nilai berada di salah satu range yang ada di dalam multi-range.

Contoh penggunaannya:

```sql
-- Cek apakah 4 ada dalam multi-range
SELECT '{[3,7), [8,9)}'::INT4MULTIRANGE @> 4;
-- Output: true (4 ada di range [3,7))

-- Cek apakah 7 ada dalam multi-range
SELECT '{[3,7), [8,9)}'::INT4MULTIRANGE @> 7;
-- Output: false (7 excluded dari [3,7))

-- Cek apakah 8 ada dalam multi-range
SELECT '{[3,7), [8,9)}'::INT4MULTIRANGE @> 8;
-- Output: true (8 ada di range [8,9))

-- Cek apakah 9 ada dalam multi-range
SELECT '{[3,7), [8,9)}'::INT4MULTIRANGE @> 9;
-- Output: false (9 excluded dari [8,9))
```

Hasilnya intuitif: sebuah nilai dianggap “termasuk” jika ia masuk ke salah satu range, dan dianggap “tidak termasuk” jika tidak masuk ke range mana pun.

#### Use Case Multi-Range

Multi-range sangat berguna ketika kamu ingin menyimpan **jadwal, periode, atau slot waktu** yang tidak berurutan dalam satu kolom. Beberapa contoh nyata:

- **Jam kerja kantor**
  Misalnya kantor buka pukul 08:00–12:00 dan kembali buka 13:00–17:00. Ada jeda 1 jam untuk istirahat makan siang.
- **Availability**
  Misalnya seseorang hanya tersedia hari Senin–Rabu dan Jumat–Sabtu.
- **Periode diskon**
  Diskon berlaku dalam dua atau tiga periode tertentu dalam setahun, misalnya Januari–Februari dan Juni–Agustus.

Multi-range membantu menyimpan semua interval itu dalam satu value yang bersih dan mudah diproses.

#### Diagram Visual Multi-Range

```
Single Range:     [========3---4---5---6---7========)
                                     GAP!
Multi-Range:      [====3---6====)   [==8==)
                   ↑ range 1 ↑       ↑ range 2 ↑
```

Diagram ini menunjukkan bagaimana sebuah multi-range dapat menyimpan dua interval yang terpisah, dengan adanya celah (gap) yang jelas di antaranya.

Dengan memahami multi-range, kamu bisa membangun logika yang lebih fleksibel untuk memodelkan data yang tidak linier atau tidak berurutan secara alami.

### j. Tabel Lengkap Range Types

#### Ringkasan Range Types di PostgreSQL

PostgreSQL menyediakan berbagai jenis **range types** yang masing-masing memiliki pasangan **multi-range types**. Setiap range memiliki subtype tertentu (misalnya integer, bigint, date, atau timestamp), dan masing-masing subtype juga memiliki kategori: apakah ia **discrete** atau **continuous**.

Memahami perbedaan ini penting karena memengaruhi bagaimana PostgreSQL menangani batas atas dan bawah, termasuk cara menginterpretasikan inclusive/exclusive pada tiap rentang. Misalnya, range dengan tipe discrete seperti `INT4RANGE` atau `DATERANGE` akan memiliki perilaku berbeda jika dibandingkan dengan tipe continuous seperti `NUMRANGE` atau `TSRANGE`.

Tabel berikut memberi gambaran lengkap mengenai tipe range dan multi-range yang tersedia:

| Range Type  | Multi-Range Type | Subtype             | Category   |
| ----------- | ---------------- | ------------------- | ---------- |
| `INT4RANGE` | `INT4MULTIRANGE` | `integer`           | Discrete   |
| `INT8RANGE` | `INT8MULTIRANGE` | `bigint`            | Discrete   |
| `NUMRANGE`  | `NUMMULTIRANGE`  | `numeric`           | Continuous |
| `TSRANGE`   | `TSMULTIRANGE`   | `timestamp`         | Continuous |
| `TSTZRANGE` | `TSTZMULTIRANGE` | `timestamp with tz` | Continuous |
| `DATERANGE` | `DATEMULTIRANGE` | `date`              | Discrete   |

#### Cara Membacanya

- **Range Type**
  Tipe data utama yang menyimpan rentang tunggal, misalnya `[1,10)`.

- **Multi-Range Type**
  Tipe data yang bisa menyimpan beberapa rentang sekaligus dalam satu kolom.

- **Subtype**
  Tipe dasar yang digunakan pada range tersebut.
  Misalnya `INT4RANGE` menggunakan `integer`, dan `TSRANGE` menggunakan `timestamp`.

- **Category**
  Menentukan apakah tipe tersebut **discrete** (berbasis nilai terpisah) atau **continuous** (rentangnya bersifat kontinu).
  Contoh:

  - `INT4RANGE` adalah discrete karena integer tidak memiliki nilai "di antara" dua angka.
  - `NUMRANGE` adalah continuous karena numeric bisa memiliki nilai pecahan tak terbatas.

Tabel ini menjadi referensi cepat ketika kamu ingin memilih tipe range yang tepat untuk kasus tertentu—misalnya ketika mengatur jadwal (timestamp), periode diskon (date), atau interval angka (integer).

### k. Use Case: Hotel Room Reservations

#### Mengapa kasus ini penting?

Dalam sistem reservasi hotel, hal yang paling krusial adalah memastikan **dua tamu tidak memesan kamar yang sama pada tanggal yang saling bertumpukan**. Jika overlap tidak dicegah secara otomatis, data reservasi bisa kacau, dan hotel bisa double-booking kamar — sesuatu yang tentu harus dihindari.

PostgreSQL menyediakan solusi elegan untuk masalah ini melalui kombinasi **range types** dan **exclusion constraints**. Dengan ini, database memastikan aturan “tidak boleh overlap” dijaga secara konsisten, tanpa perlu logika manual tambahan di sisi aplikasi.

#### Implementasi dengan Range dan Exclusion Constraint

Berikut contoh tabel reservasi yang memanfaatkan `DATERANGE` untuk menyimpan periode menginap (`stay_period`), dan `EXCLUDE USING GIST` untuk mencegah dua reservasi yang overlap pada kamar yang sama.

```sql
CREATE TABLE reservations (
  id SERIAL PRIMARY KEY,
  room_id INT,
  guest_name TEXT,
  stay_period DATERANGE,
  -- Exclusion constraint: cegah overlap untuk room yang sama
  EXCLUDE USING GIST (
    room_id WITH =,
    stay_period WITH &&
  )
);
```

Constraint di atas membuat PostgreSQL menolak setiap insert atau update yang menyebabkan dua kondisi berikut terjadi secara bersamaan:

1. **room_id sama**, dan
2. **periode stay_period overlap** (`&&`).

Jika kedua kondisi terpenuhi, database langsung menolak operasi tersebut.

#### Contoh Perilaku Constraint

1. **Reservasi pertama: valid**

```sql
INSERT INTO reservations (room_id, guest_name, stay_period)
VALUES (101, 'Alice', '[2025-01-01,2025-01-05)'::DATERANGE);
```

Tidak ada konflik, sehingga data masuk tanpa masalah.

2. **Reservasi overlap di kamar yang sama: ditolak**

```sql
INSERT INTO reservations (room_id, guest_name, stay_period)
VALUES (101, 'Bob', '[2025-01-03,2025-01-07)'::DATERANGE);
```

Hasil:

```
ERROR: conflicting key value violates exclusion constraint
```

Karena tanggal 3–5 Januari overlap dengan reservasi sebelumnya milik Alice.

3. **Reservasi overlap tapi di kamar berbeda: diterima**

```sql
INSERT INTO reservations (room_id, guest_name, stay_period)
VALUES (102, 'Bob', '[2025-01-03,2025-01-07)'::DATERANGE);
```

Walaupun tanggalnya overlap, kamar berbeda → tidak melanggar constraint.

4. **Reservasi setelah check-out: valid**

```sql
INSERT INTO reservations (room_id, guest_name, stay_period)
VALUES (101, 'Charlie', '[2025-01-05,2025-01-10)'::DATERANGE);
```

Karena range pertama adalah `[2025-01-01, 2025-01-05)` yang berarti **tanggal 5 tidak termasuk**, maka booking berikutnya boleh dimulai tepat pada hari itu.

#### Cara Kerja Exclusion Constraint

Agar lebih mudah dipahami:

- `room_id WITH =`
  PostgreSQL hanya akan membandingkan record jika `room_id` sama.

- `stay_period WITH &&`
  PostgreSQL mengecek apakah rentang tanggal saling tumpang tindih.

- Jika **keduanya benar**, maka penyimpanan data **ditolak otomatis**.

Konsep ini membuat sistem reservasi menjadi jauh lebih aman dan andal. Developer tidak perlu menulis logic manual untuk memastikan periode tidak bentrok — database yang mengerjakannya untuk kita.

---

## 3. Hubungan Antar Konsep

```
┌──────────────────────────────────────────────┐
│         RANGE META TYPE                      │
└──────────────────────────────────────────────┘
                    │
        ┌───────────┴────────────┐
        │                        │
   ┌────▼────┐              ┌────▼─────┐
   │ Discrete│              │Continuous│
   │ Ranges  │              │ Ranges   │
   └────┬────┘              └────┬─────┘
        │                        │
   - INT4RANGE              - NUMRANGE
   - INT8RANGE              - TSRANGE
   - DATERANGE              - TSTZRANGE
        │                        │
        └───────────┬────────────┘
                    │
        ┌───────────▼────────────┐
        │   BOUNDS (inclusive/   │
        │   exclusive, unbounded)│
        └───────────┬────────────┘
                    │
        ┌───────────┴────────────┐
        │                        │
   ┌────▼────┐              ┌────▼──────┐
   │ Single  │              │Multi-Range│
   │ Range   │              │(multiple  │
   └────┬────┘              │ ranges)   │
        │                   └────┬──────┘
        │                        │
        └───────────┬────────────┘
                    │
        ┌───────────▼────────────┐
        │    OPERATORS           │
        ├────────────────────────┤
        │ @>  (containment)      │
        │ &&  (overlap)          │
        │ *   (intersection)     │
        │ UPPER/LOWER functions  │
        └────────────────────────┘
```

**Alur pemahaman:**

1. **Range adalah meta type** dengan berbagai subtype (int, numeric, date, timestamp)

2. **Discrete vs Continuous**:

   - Discrete: ada step unit → PostgreSQL normalize bounds
   - Continuous: infinite values → bounds tidak di-normalize

3. **Bounds menentukan inklusi**:

   - `[` `]` = inclusive (nilai termasuk)
   - `(` `)` = exclusive (nilai tidak termasuk)
   - Bisa unbounded (tanpa batas)
   - Bisa empty (tidak ada range)

4. **Single range vs Multi-range**:

   - Single: satu rentang
   - Multi: beberapa rentang

5. **Operators untuk query**:

   - Containment: apakah nilai ada dalam range?
   - Overlap: apakah dua range tumpang tindih?
   - Intersection: irisan dua range
   - Upper/Lower: ambil bounds programmatically

6. **Use case praktis**:
   - Reservasi hotel (cegah overlap)
   - Price tiers berdasarkan quantity
   - Access control based on time periods
   - Historical data versioning

---

## 4. Kesimpulan

Range types adalah fitur powerful PostgreSQL untuk merepresentasikan dan query data **rentang nilai**. Dengan range, kita bisa:

**Key features:**

- **Flexible bounds**: inclusive, exclusive, atau unbounded
- **Multiple subtypes**: integer, numeric, date, timestamp
- **Powerful operators**: containment, overlap, intersection
- **Multi-range support**: untuk rentang non-contiguous
- **Constraint enforcement**: cegah overlap dengan exclusion constraints

**Kapan menggunakan ranges:**

- **Reservasi/booking systems**: hotel, meeting room, equipment rental
- **Temporal data**: historical records, validity periods
- **Pricing tiers**: quantity discounts, membership levels
- **Access control**: time-based permissions, blackout periods

**Perbedaan penting:**

- **Discrete ranges** (int, date): PostgreSQL normalize bounds untuk konsistensi
- **Continuous ranges** (numeric, timestamp): bounds tetap seperti input
- **Empty** ≠ **unbounded**: empty = tidak ada range, unbounded = infinite range

**Complexity level:**

```
Basic:        Single range dengan fixed bounds
Intermediate: Unbounded ranges, overlap detection
Advanced:     Multi-ranges, exclusion constraints
```

Range types mungkin terlihat kompleks di awal, tapi setelah memahami konsep **discrete vs continuous** dan **inclusive vs exclusive**, semuanya akan jadi lebih jelas. Practice dengan berbagai contoh akan sangat membantu!
