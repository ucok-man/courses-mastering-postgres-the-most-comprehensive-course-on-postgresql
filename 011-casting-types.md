# 📝 Casting Types di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas berbagai cara melakukan casting (konversi tipe data) di PostgreSQL, serta beberapa fungsi utility yang berguna untuk debugging dan optimasi. Materi mencakup perbedaan antara syntax casting PostgreSQL dengan SQL standar, fungsi untuk memeriksa tipe data dan ukuran kolom, serta konsep decorated literal. Video juga membahas perdebatan tentang portabilitas database dan memberikan tips praktis untuk memilih tipe data yang tepat.

## 2. Konsep Utama

### a. Cara Melakukan Casting

Casting adalah proses mengubah suatu nilai dari satu tipe data ke tipe data lain. Di PostgreSQL, casting sangat penting karena menentukan bagaimana sebuah nilai diperlakukan oleh database, misalnya apakah sebuah angka dianggap sebagai bilangan bulat, uang, atau tipe numerik lain. PostgreSQL menyediakan beberapa cara untuk melakukan casting, masing-masing dengan gaya dan kegunaan yang berbeda.

Secara umum, ada tiga pendekatan utama yang bisa digunakan untuk melakukan casting di PostgreSQL.

#### 1. PostgreSQL-specific syntax (double colon `::`)

Syntax ini adalah cara paling umum dan paling sering digunakan oleh pengguna PostgreSQL. Bentuknya ringkas dan mudah dibaca, sehingga sangat populer dalam query sehari-hari.

```sql
SELECT 100::MONEY;
SELECT 100::INT8;
SELECT 100::INT4;
```

Pada contoh di atas, literal `100` awalnya dianggap sebagai nilai numerik tanpa tipe yang spesifik. Dengan menambahkan `::MONEY`, PostgreSQL diminta untuk memperlakukan nilai tersebut sebagai tipe `MONEY`, sehingga hasilnya akan ditampilkan dalam format mata uang (misalnya `$100.00`, tergantung locale).

Sementara itu, `::INT8` mengubah nilai menjadi integer 8-byte (bigint), dan `::INT4` mengubahnya menjadi integer 4-byte (integer biasa). Perbedaannya tidak terlihat dari nilai `100` itu sendiri, tetapi sangat penting untuk penyimpanan data dan operasi lanjutan.

#### 2. SQL Standard syntax (CAST function)

Cara kedua menggunakan fungsi `CAST`, yang merupakan bagian dari standar SQL. Keunggulan utama pendekatan ini adalah sifatnya yang lebih portable, artinya query ini lebih mudah dipindahkan ke database lain yang juga mengikuti standar SQL.

```sql
SELECT CAST(100 AS MONEY);
SELECT CAST(100 AS INT8);
SELECT CAST(100 AS INT4);
```

Secara konsep dan hasil, pendekatan ini sama dengan penggunaan `::`. Perbedaannya hanya pada gaya penulisan. PostgreSQL akan menghasilkan output yang identik, hanya saja sintaks ini biasanya terasa lebih panjang dan kurang ringkas dibandingkan `::`. Dalam praktik, banyak developer PostgreSQL tetap memilih `::`, kecuali jika memang menargetkan kompatibilitas lintas database.

#### 3. Decorated Literal

Pendekatan ketiga adalah menggunakan decorated literal, yaitu menuliskan tipe data langsung di depan literalnya.

```sql
SELECT INTEGER '100';
-- Hasilnya adalah INT4
```

Pada contoh ini, PostgreSQL langsung mengetahui bahwa literal `'100'` harus diperlakukan sebagai `INTEGER` (yang secara internal adalah `INT4`). Cara ini memberi “hint” eksplisit kepada PostgreSQL tentang tipe data literal tersebut sejak awal.

Namun, syntax ini jarang digunakan dalam praktik sehari-hari karena tidak konsisten untuk semua tipe data dan dokumentasinya relatif terbatas.

#### Perbandingan hasil casting

Untuk memahami bahwa ketiga pendekatan di atas pada dasarnya melakukan hal yang sama, perhatikan contoh berikut:

```sql
SELECT 100::MONEY;
SELECT CAST(100 AS MONEY);
```

Kedua query tersebut menghasilkan output yang sama, yaitu nilai uang `$100.00`. Ini menunjukkan bahwa perbedaannya hanya pada cara penulisan, bukan pada hasil akhirnya.

#### Catatan penting tentang Decorated Literal

Meskipun decorated literal bisa berguna dalam situasi tertentu, ada beberapa hal penting yang perlu diperhatikan:

- Tidak semua tipe data mendukung syntax ini.
- Misalnya, penulisan `INT8 '100'` tidak akan bekerja. Untuk integer 4-byte, yang valid adalah `INTEGER '100'`.
- Dokumentasinya tidak selengkap dua metode sebelumnya dan jarang digunakan dalam kode produksi.
- Fungsi utamanya adalah memberi petunjuk awal kepada PostgreSQL tentang tipe data literal, bukan sebagai pengganti utama mekanisme casting lain.

Karena alasan-alasan tersebut, dalam praktik sehari-hari, kebanyakan developer PostgreSQL lebih memilih menggunakan `::` atau `CAST`, dan hanya menggunakan decorated literal dalam kasus yang sangat spesifik.

### b. Fungsi `pg_typeof()` - Memeriksa Tipe Data

Dalam praktik sehari-hari, terutama saat menulis query yang cukup kompleks, kita sering berhadapan dengan nilai yang _terlihat_ sama tetapi sebenarnya memiliki tipe data yang berbeda. Di sinilah fungsi `pg_typeof()` menjadi sangat berguna. Fungsi ini membantu kita memastikan tipe data sebenarnya dari suatu ekspresi atau nilai yang dihasilkan oleh PostgreSQL.

#### Mengapa `pg_typeof()` penting

Perhatikan contoh sederhana berikut:

```sql
-- Membandingkan dua tipe data yang terlihat sama
SELECT 100::INT8, 100::INT4;
-- Output: 100, 100 (sulit dibedakan!)
```

Secara visual, kedua kolom di atas menampilkan angka `100`. Dari hasil ini saja, kita tidak bisa mengetahui bahwa kolom pertama bertipe `INT8` (bigint) dan kolom kedua bertipe `INT4` (integer). Padahal, perbedaan tipe ini bisa berdampak pada operasi lanjutan, performa, maupun kompatibilitas dengan fungsi atau kolom lain di database.

#### Melihat tipe data sebenarnya dengan `pg_typeof()`

Untuk mengatasi kebingungan tersebut, PostgreSQL menyediakan fungsi `pg_typeof()`. Fungsi ini mengembalikan nama tipe data dari ekspresi yang diberikan.

```sql
-- Menggunakan pg_typeof untuk melihat perbedaannya
SELECT pg_typeof(100::INT8);  -- bigint
SELECT pg_typeof(100::INT4);  -- integer
```

Dari hasil ini, kita bisa melihat dengan jelas bahwa meskipun nilainya sama, tipe datanya berbeda. `INT8` dipetakan ke `bigint`, sedangkan `INT4` dipetakan ke `integer`.

#### Memastikan konsistensi tipe data

`pg_typeof()` juga sangat membantu ketika kita ingin memastikan bahwa beberapa ekspresi atau kolom memiliki tipe data yang konsisten.

```sql
-- Memastikan kedua kolom memiliki tipe yang sama
SELECT
    pg_typeof(100::INT8) AS tipe1,  -- bigint
    pg_typeof(100::INT8) AS tipe2;  -- bigint
```

Pada contoh di atas, kedua kolom secara eksplisit di-cast ke `INT8`, sehingga `pg_typeof()` mengonfirmasi bahwa keduanya memang bertipe `bigint`. Teknik seperti ini sering digunakan untuk memverifikasi hasil query sebelum dipakai lebih lanjut, misalnya sebelum dimasukkan ke tabel atau diproses oleh fungsi lain.

#### Kapan menggunakan `pg_typeof()`

Fungsi `pg_typeof()` paling bermanfaat dalam beberapa situasi berikut:

- Saat melakukan debugging pada query yang kompleks, terutama yang melibatkan banyak operasi atau casting.
- Untuk memverifikasi tipe data output dari sebuah fungsi atau ekspresi.
- Ketika tidak yakin tipe data apa yang dihasilkan oleh suatu operasi, meskipun nilainya terlihat benar secara visual.

Dengan memanfaatkan `pg_typeof()`, kita bisa lebih percaya diri bahwa query yang kita tulis tidak hanya menghasilkan nilai yang benar, tetapi juga tipe data yang tepat.

### c. Fungsi `pg_column_size()` - Memeriksa Ukuran Data

Selain memahami tipe data, penting juga untuk mengetahui berapa besar ruang penyimpanan yang digunakan oleh suatu nilai. PostgreSQL menyediakan fungsi `pg_column_size()` untuk tujuan ini. Fungsi ini mengembalikan ukuran data dalam satuan byte, yaitu berapa byte yang benar-benar digunakan untuk menyimpan sebuah nilai di dalam database.

Pemahaman tentang ukuran data ini sangat berguna ketika kita ingin mengoptimalkan skema database, terutama pada tabel dengan jumlah data yang besar.

#### Memeriksa ukuran berbagai tipe integer

Perhatikan contoh berikut:

```sql
-- Memeriksa ukuran berbagai tipe integer
SELECT pg_column_size(100::INT2);  -- 2 bytes
SELECT pg_column_size(100::INT4);  -- 4 bytes
SELECT pg_column_size(100::INT8);  -- 8 bytes
```

Meskipun ketiga nilai tersebut sama-sama menyimpan angka `100`, hasil dari `pg_column_size()` menunjukkan ukuran yang berbeda. Hal ini terjadi karena setiap tipe integer di PostgreSQL memiliki ukuran penyimpanan yang sudah ditentukan sejak awal.

Pelajaran penting di sini adalah: nilai yang sama bisa menggunakan ruang penyimpanan yang berbeda, tergantung tipe data yang dipilih.

#### Integer: ukuran bersifat fixed

Untuk tipe integer, ukuran penyimpanan bersifat tetap (fixed). Artinya, besar kecilnya nilai tidak memengaruhi ukuran data yang digunakan. Selama tipenya sama, ukuran penyimpanannya akan selalu sama.

```sql
-- Untuk integer, ukuran ditentukan oleh tipe kolom, bukan nilai
SELECT
    pg_column_size(100::INT2) AS small,      -- 2 bytes
    pg_column_size(100::INT4) AS medium,     -- 4 bytes
    pg_column_size(100::INT8) AS large;      -- 8 bytes
-- Semua menyimpan nilai 100, tapi ukuran berbeda!
```

Dari contoh di atas, terlihat jelas bahwa `INT2`, `INT4`, dan `INT8` masing-masing selalu menggunakan 2, 4, dan 8 byte. Baik nilainya kecil maupun besar (selama masih dalam rentang yang valid), ukuran penyimpanannya tidak akan berubah.

Karakteristik ini membuat tipe integer sangat efisien dan mudah diprediksi dari sisi penggunaan storage.

#### Numeric: ukuran bersifat variable

Berbeda dengan integer, tipe `NUMERIC` memiliki ukuran penyimpanan yang bersifat variabel. Ukuran data akan bertambah seiring dengan meningkatnya jumlah digit dan presisi yang dibutuhkan.

```sql
-- Untuk numeric, ukuran berubah sesuai nilai yang disimpan
SELECT pg_column_size(100::NUMERIC);                 -- 8 bytes
SELECT pg_column_size(10000.123456::NUMERIC);        -- 14 bytes
SELECT pg_column_size(10000.123456789012::NUMERIC);  -- lebih besar lagi
```

Pada contoh pertama, nilai `100` sebagai `NUMERIC` hanya membutuhkan 8 byte. Namun, ketika jumlah digit dan angka di belakang koma bertambah, ukuran penyimpanan juga ikut meningkat. Semakin tinggi presisi yang diperlukan, semakin besar pula ruang yang digunakan.

Inilah alasan mengapa `NUMERIC` sangat fleksibel dan akurat, tetapi dari sisi penyimpanan dan performa, ia lebih “mahal” dibandingkan tipe integer. Oleh karena itu, pemilihan antara integer dan numeric sebaiknya disesuaikan dengan kebutuhan presisi dan skala data yang benar-benar diperlukan.

### d. Perdebatan: Portabilitas vs Fitur Spesifik

Bagian ini membahas perdebatan klasik dalam dunia database: apakah sebaiknya kita menulis SQL yang sepenuhnya mengikuti standar agar mudah dipindahkan ke database lain, atau memanfaatkan fitur spesifik dari database yang kita gunakan untuk mendapatkan kenyamanan dan kejelasan yang lebih baik. Video ini mengambil posisi yang cukup praktis dan realistis terhadap dilema tersebut.

#### Argumen untuk SQL Standard (`CAST`)

Pendekatan pertama menekankan pentingnya portabilitas. Dengan menggunakan syntax standar SQL seperti `CAST`, query yang kita tulis lebih mudah dijalankan di berbagai sistem database.

Beberapa alasan utama mendukung pendekatan ini:

- Query lebih portable dan bisa dijalankan di database lain seperti MySQL, SQL Server, atau Oracle tanpa banyak perubahan.
- Sangat cocok untuk library, ORM, atau framework yang memang dirancang untuk mendukung banyak jenis database.
- Mengikuti standar industri, sehingga lebih aman dari sudut pandang kompatibilitas jangka panjang.

Dalam konteks ini, `CAST` dianggap sebagai pilihan yang “aman” karena tidak mengikat kode kita pada satu vendor database tertentu.

#### Argumen untuk PostgreSQL Syntax (`::`)

Di sisi lain, PostgreSQL menyediakan syntax casting `::` yang lebih ringkas dan ekspresif. Pendukung pendekatan ini berargumen bahwa jika aplikasi memang sudah menggunakan PostgreSQL, tidak ada alasan kuat untuk menghindari fitur yang disediakan secara native.

Argumen yang sering diajukan antara lain:

- Penulisan lebih singkat dan lebih mudah dibaca, terutama pada query yang kompleks.
- Jika proyek sudah berkomitmen menggunakan PostgreSQL, membatasi diri pada standar umum justru bisa membuat kode kurang optimal.
- Dalam praktik nyata, perpindahan database besar jarang terjadi, karena biayanya tinggi dan kompleks.

Dengan kata lain, menggunakan `::` dianggap sebagai keputusan yang realistis dan sesuai dengan kebutuhan sehari-hari.

#### Rekomendasi dari video

Alih-alih mengambil posisi ekstrem, video ini memberikan rekomendasi yang cukup seimbang dan kontekstual:

- Untuk aplikasi yang secara spesifik dibangun di atas PostgreSQL, gunakan syntax `::` karena lebih ringkas dan nyaman.
- Untuk library atau framework yang memang ditujukan untuk mendukung banyak database, gunakan `CAST` agar tetap portable.
- Jangan mengorbankan fitur atau kenyamanan hanya karena kekhawatiran “mungkin suatu hari pindah database”, jika secara realistis hal tersebut kecil kemungkinannya.

#### Contoh perbandingan dalam kode

Perbedaan kedua pendekatan ini bisa dilihat dari contoh berikut:

```sql
-- Pilihan 1: PostgreSQL style (lebih ringkas)
SELECT amount::NUMERIC(10,2) FROM transactions;

-- Pilihan 2: Standard SQL (lebih portable)
SELECT CAST(amount AS NUMERIC(10,2)) FROM transactions;
```

Kedua query di atas menghasilkan hasil yang sama: kolom `amount` diperlakukan sebagai `NUMERIC(10,2)`. Perbedaannya hanya pada gaya penulisan dan tingkat portabilitas. Pilihan terbaik bukan soal benar atau salah, melainkan soal konteks dan tujuan dari aplikasi atau sistem yang sedang dibangun.

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

## 4. Kesimpulan

Casting di PostgreSQL bisa dilakukan dengan tiga cara: syntax PostgreSQL (`::`, ringkas), SQL Standard (`CAST`, portable), dan decorated literal (jarang). Untuk debugging dan optimasi, PostgreSQL menyediakan fungsi utility:

- `pg_typeof()` untuk mengecek tipe data
- `pg_column_size()` untuk mengecek ukuran storage

**Prinsip utama:** Pilih tipe data terkecil yang bisa menampung data Anda. Ini penting karena tipe data menentukan ukuran storage (terutama untuk integer dengan ukuran fixed). Numeric memiliki ukuran variable yang tumbuh sesuai presisi data.

**Rekomendasi praktis:** Jika aplikasi Anda khusus menggunakan PostgreSQL, gunakan fitur-fitur PostgreSQL secara penuh (termasuk `::` syntax). Jangan membatasi diri hanya karena kemungkinan portabilitas di masa depan yang mungkin tidak pernah terjadi. Namun, untuk library atau framework, pertimbangkan SQL standard untuk kompatibilitas lebih luas.

**Key takeaway:** Kombinasikan `pg_typeof()` dan `pg_column_size()` untuk membuat keputusan yang tepat dalam schema design dan optimasi database.
