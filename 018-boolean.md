# 📝 Tipe Data Boolean di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang tipe data Boolean di PostgreSQL. Meskipun terlihat sederhana (hanya true/false), ada beberapa hal menarik tentang implementasi Boolean di Postgres. Berbeda dengan database lain yang menggunakan integer kecil sebagai alias, PostgreSQL memiliki tipe data Boolean yang sesungguhnya dengan ukuran hanya 1 byte dan dapat menerima berbagai format input yang akan otomatis dikonversi.

## 2. Konsep Utama

### a. Boolean sebagai Tipe Data Native di PostgreSQL

PostgreSQL menyediakan **tipe data Boolean yang benar-benar native**, artinya Boolean di Postgres bukan sekadar representasi angka seperti `0` atau `1`. Ini berbeda dengan beberapa sistem basis data lain yang di balik layar sebenarnya menyimpan nilai Boolean sebagai _tiny integer_ untuk memenuhi standar SQL.

Pendekatan ini sejalan dengan filosofi PostgreSQL yang cukup terkenal: _“there is a data type for everything”_. Jika suatu konsep memang layak menjadi tipe data tersendiri, maka Postgres akan menyediakannya secara eksplisit—termasuk untuk nilai benar dan salah.

Dengan adanya tipe data Boolean native, Postgres dapat:

- Menjaga konsistensi data dengan lebih baik
- Membuat skema database lebih jelas secara semantik
- Mengurangi ambiguitas dibandingkan penggunaan angka sebagai pengganti Boolean

#### Contoh Deklarasi Kolom Boolean

Berikut contoh sederhana pembuatan tabel dengan kolom bertipe Boolean:

```sql
CREATE TABLE boolean_example (
    status BOOLEAN
);
```

Pada contoh di atas, kolom `status` secara eksplisit didefinisikan sebagai `BOOLEAN`. Ini memberi tahu PostgreSQL bahwa kolom tersebut hanya dimaksudkan untuk menyimpan nilai logis, bukan angka atau teks.

#### Contoh Penggunaan dan Nilai yang Didukung

PostgreSQL cukup fleksibel dalam menerima input untuk Boolean. Beberapa nilai yang valid antara lain:

- `TRUE`, `FALSE`
- `true`, `false`
- `t`, `f`
- `1`, `0`
- `yes`, `no` (dalam konteks tertentu)

Contoh memasukkan data:

```sql
INSERT INTO boolean_example (status) VALUES (TRUE);
INSERT INTO boolean_example (status) VALUES (FALSE);
```

Jika kita melakukan query:

```sql
SELECT status FROM boolean_example;
```

Hasilnya akan ditampilkan sebagai nilai Boolean yang jelas (`t` atau `f`), bukan angka. Ini menegaskan bahwa Postgres memang memperlakukan Boolean sebagai tipe data logis, bukan sekadar angka yang “dipinjam” untuk mewakili kondisi tertentu.

Dengan cara ini, struktur database menjadi lebih mudah dibaca dan dipahami, terutama saat bekerja dalam tim atau ketika skema database berkembang menjadi semakin kompleks.

### b. Tiga Kemungkinan Nilai Boolean

Di PostgreSQL, tipe data Boolean tidak hanya mengenal dua kondisi seperti yang sering kita temui dalam logika dasar. Selain `TRUE` dan `FALSE`, PostgreSQL juga mendukung nilai ketiga, yaitu `NULL`. Inilah yang membuat Boolean di Postgres lebih ekspresif dan realistis dalam merepresentasikan kondisi data di dunia nyata.

Secara konseptual, ketiga nilai Boolean tersebut memiliki makna sebagai berikut:

- **`TRUE`**
  Menunjukkan kondisi benar atau terpenuhi. Nilai ini digunakan ketika suatu pernyataan atau status secara eksplisit dinyatakan benar.

- **`FALSE`**
  Menunjukkan kondisi salah atau tidak terpenuhi. Nilai ini digunakan ketika suatu pernyataan secara jelas dinyatakan tidak benar.

- **`NULL`**
  Menunjukkan kondisi _tidak diketahui_ (_unknown state_). Nilai ini **bukan berarti salah**, melainkan menandakan bahwa informasi tentang status tersebut belum tersedia, belum ditentukan, atau memang tidak diketahui.

#### Mengapa `NULL` Penting dalam Boolean

Keberadaan `NULL` sangat penting untuk merepresentasikan situasi nyata yang sering terjadi dalam sistem informasi. Misalnya:

- Status suatu proses belum diputuskan
- Data belum diinput oleh pengguna
- Informasi memang tidak tersedia pada saat tertentu

Dalam kasus-kasus tersebut, memaksa nilai menjadi `TRUE` atau `FALSE` justru bisa menyesatkan. Dengan `NULL`, database dapat menyimpan fakta bahwa status tersebut masih belum jelas.

#### Contoh Penggunaan dalam Tabel

Misalkan kita memiliki tabel dengan kolom Boolean:

```sql
CREATE TABLE approval_example (
    approved BOOLEAN
);
```

Kita bisa mengisi kolom `approved` dengan tiga kemungkinan nilai:

```sql
INSERT INTO approval_example (approved) VALUES (TRUE);
INSERT INTO approval_example (approved) VALUES (FALSE);
INSERT INTO approval_example (approved) VALUES (NULL);
```

Jika kita melakukan query:

```sql
SELECT approved FROM approval_example;
```

Hasilnya akan memperlihatkan tiga jenis nilai yang berbeda: `t`, `f`, dan `NULL`. Ini menunjukkan bahwa PostgreSQL benar-benar memperlakukan `NULL` sebagai kondisi tersendiri, bukan sekadar nilai default atau pengganti `FALSE`.

Dengan memahami tiga kemungkinan nilai ini, kita dapat merancang skema dan logika aplikasi yang lebih akurat, terutama saat berhadapan dengan data yang bersifat belum lengkap atau masih dalam proses.

### c. Fleksibilitas Input: Berbagai Format yang Diterima

PostgreSQL sangat fleksibel dalam menerima input untuk tipe Boolean. Berikut format-format yang dapat diterima:

| Format Input                    | Hasil Konversi |
| ------------------------------- | -------------- |
| `true`, `'true'`, `'t'`         | `TRUE`         |
| `false`, `'false'`, `'f'`       | `FALSE`        |
| `'1'` (string), `'on'`, `'yes'` | `TRUE`         |
| `'0'` (string), `'off'`, `'no'` | `FALSE`        |
| `NULL`                          | `NULL`         |

**Contoh insert dengan berbagai format:**

```sql
INSERT INTO boolean_example (status) VALUES
    (true),           -- boolean literal
    (false),          -- boolean literal
    ('t'),            -- string 't'
    ('true'),         -- string 'true'
    ('f'),            -- string 'f'
    ('false'),        -- string 'false'
    ('1'),            -- string '1'
    ('0'),            -- string '0'
    ('on'),           -- string 'on'
    ('off'),          -- string 'off'
    ('yes'),          -- string 'yes'
    ('no'),           -- string 'no'
    (NULL);           -- null value
```

**Hasil query:**

```sql
SELECT * FROM boolean_example;
```

Semua nilai di atas akan dikonversi dan ditampilkan sebagai `t` (true), `f` (false), atau `null`.

### d. Casting Eksplisit vs Implicit

Dalam PostgreSQL, terdapat perbedaan perilaku yang cukup penting antara **casting secara eksplisit** (_explicit cast_) dan **memasukkan nilai secara langsung** (_implicit cast saat insert_), khususnya ketika bekerja dengan tipe data Boolean. Perbedaan ini sering kali menjadi sumber kebingungan jika belum memahami cara Postgres memproses tipe data.

#### Explicit Cast: Konversi yang Disengaja dan Aman

**Explicit cast** berarti kita secara sadar dan eksplisit memberitahu PostgreSQL untuk mengubah suatu nilai dari satu tipe data ke tipe data lain. Hal ini biasanya dilakukan dengan operator `::`.

Contoh berikut **berfungsi dengan baik**:

```sql
SELECT 1::BOOLEAN;        -- menggunakan integer literal
SELECT '1'::BOOLEAN;      -- menggunakan string
```

Pada contoh di atas:

- `1::BOOLEAN` secara eksplisit meminta PostgreSQL untuk mengonversi integer `1` menjadi nilai Boolean.
- `'1'::BOOLEAN` melakukan hal yang sama, tetapi menggunakan string sebagai input.

Karena instruksinya jelas, PostgreSQL akan melakukan konversi:

- `1` atau `'1'` → `TRUE`
- `0` atau `'0'` → `FALSE`

Hasil dari query tersebut akan berupa nilai Boolean (`t` atau `f`), bukan angka.

#### Insert Langsung: Tidak Ada Implicit Cast dari Integer ke Boolean

Berbeda dengan `SELECT`, saat kita melakukan `INSERT`, PostgreSQL **tidak melakukan implicit cast** dari integer ke Boolean. Contoh berikut akan menghasilkan error:

```sql
-- Ini akan ERROR:
INSERT INTO boolean_example (status) VALUES (1);
```

Error ini terjadi karena:

- Kolom `status` bertipe `BOOLEAN`
- Nilai `1` bertipe `INTEGER`
- PostgreSQL tidak menyediakan konversi otomatis (implicit cast) dari `INTEGER` ke `BOOLEAN` dalam konteks `INSERT`

Postgres sengaja bersikap ketat untuk mencegah ambiguitas dan potensi kesalahan data.

#### Cara yang Benar Saat Insert Data Boolean

Jika ingin memasukkan nilai yang merepresentasikan Boolean, gunakan format yang memang diakui sebagai Boolean, misalnya:

```sql
INSERT INTO boolean_example (status) VALUES (TRUE);
INSERT INTO boolean_example (status) VALUES (FALSE);
INSERT INTO boolean_example (status) VALUES ('1');
INSERT INTO boolean_example (status) VALUES ('0');
```

Dalam contoh di atas:

- `TRUE` dan `FALSE` sudah jelas merupakan nilai Boolean
- `'1'` dan `'0'` adalah string yang dapat dikonversi oleh PostgreSQL ke Boolean

#### Ringkasan Poin Penting

- **Explicit cast** (`::BOOLEAN`) memberi instruksi jelas kepada PostgreSQL untuk melakukan konversi tipe data
- PostgreSQL dapat mengonversi integer `1` atau `0` menjadi Boolean **hanya jika diminta secara eksplisit**
- Saat melakukan `INSERT`, **integer langsung tidak diterima** untuk kolom Boolean
- Gunakan nilai Boolean (`TRUE` / `FALSE`) atau string (`'1'` / `'0'`) agar data dapat disimpan dengan benar

Memahami perbedaan ini akan membantu Anda menghindari error yang tidak perlu dan membuat interaksi dengan PostgreSQL menjadi lebih konsisten dan dapat diprediksi.

### e. Ukuran Storage: Hanya 1 Byte!

Salah satu keunggulan tipe data Boolean di PostgreSQL adalah **efisiensi penyimpanan**. Meskipun terlihat sederhana, pemilihan tipe data yang tepat dapat memberikan dampak besar, terutama pada tabel dengan jumlah baris yang sangat banyak.

PostgreSQL menyimpan nilai Boolean hanya dengan **1 byte**, jauh lebih kecil dibandingkan tipe numerik yang sering dijadikan pengganti Boolean di beberapa sistem lain.

#### Contoh Pengukuran Ukuran Storage

PostgreSQL menyediakan fungsi `pg_column_size()` untuk mengetahui berapa byte yang digunakan oleh suatu nilai. Perhatikan contoh berikut:

```sql
SELECT pg_column_size(true::BOOLEAN);   -- Hasil: 1 byte
SELECT pg_column_size(1::INTEGER);      -- Hasil: 4 bytes
SELECT pg_column_size(1::SMALLINT);     -- Hasil: 2 bytes (alias: int2)
```

Penjelasan dari hasil di atas:

- `true::BOOLEAN` hanya memakan **1 byte**
- `INTEGER` membutuhkan **4 byte**
- `SMALLINT` (atau `int2`) membutuhkan **2 byte**

Ini menunjukkan bahwa Boolean adalah pilihan paling ringan secara storage untuk merepresentasikan nilai logis.

#### Perbandingan Ukuran Antar Tipe Data

Jika kita bandingkan secara langsung:

- `BOOLEAN`: **1 byte**
- `SMALLINT` (`int2`): **2 bytes** — sekitar 2 kali lebih besar
- `INTEGER`: **4 bytes** — sekitar 4 kali lebih besar

Perbedaan ini mungkin tampak kecil pada satu baris data, tetapi akan menjadi signifikan pada tabel yang berisi jutaan atau bahkan miliaran baris.

#### Mengapa Efisiensi Ini Penting

Ada beberapa alasan mengapa memilih Boolean lebih tepat dibandingkan tipe numerik untuk nilai logis:

- `SMALLINT` memang memiliki rentang nilai hingga sekitar ±32.000, tetapi untuk status logis kita hanya membutuhkan dua kondisi: benar atau salah
- Menggunakan Boolean berarti menghemat **hingga 50% storage** dibandingkan `SMALLINT`
- Lebih sedikit penggunaan storage juga berdampak positif pada performa, seperti:

  - Cache yang lebih efisien
  - I/O disk yang lebih ringan
  - Indeks yang lebih kecil dan cepat

Pada akhirnya, prinsip yang perlu dipegang adalah: **gunakan tipe data yang paling tepat untuk merepresentasikan data Anda**. Untuk nilai logis dengan dua (atau tiga, jika termasuk `NULL`) kemungkinan, Boolean adalah pilihan yang paling jelas, efisien, dan semantik.

### f. Konversi ke Tipe Data Lain

Selain digunakan sebagai tipe data logis, Boolean di PostgreSQL juga dapat **dikonversi ke tipe data lain**, seperti integer. Konversi ini berguna ketika nilai Boolean perlu diolah lebih lanjut dalam konteks numerik, misalnya untuk perhitungan, agregasi, atau integrasi dengan sistem lain yang mengharapkan angka.

#### Konversi Boolean ke Integer

PostgreSQL menyediakan mekanisme _explicit cast_ untuk mengubah nilai Boolean menjadi integer. Contohnya sebagai berikut:

```sql
-- Konversi Boolean ke integer
SELECT true::INTEGER;     -- Hasil: 1
SELECT false::INTEGER;    -- Hasil: 0
```

Pada contoh di atas:

- `true::INTEGER` dikonversi menjadi `1`
- `false::INTEGER` dikonversi menjadi `0`

Konversi ini bersifat deterministik dan konsisten, sehingga aman digunakan dalam logika aplikasi. Pola ini sering dimanfaatkan, misalnya, untuk menghitung jumlah kondisi `true` dengan fungsi agregasi seperti `SUM()`.

Sebagai contoh:

```sql
SELECT SUM(status::INTEGER) FROM boolean_example;
```

Jika kolom `status` berisi nilai `TRUE` dan `FALSE`, maka:

- Setiap `TRUE` akan dihitung sebagai `1`
- Setiap `FALSE` akan dihitung sebagai `0`

Hasilnya adalah jumlah baris dengan nilai `TRUE`.

#### Memeriksa Tipe Data Hasil Konversi

Untuk memastikan tipe data dari suatu ekspresi, PostgreSQL menyediakan fungsi `pg_typeof()`. Fungsi ini sangat berguna saat belajar atau melakukan debugging query.

Contoh penggunaannya:

```sql
SELECT pg_typeof(true::BOOLEAN);   -- Hasil: 'boolean'
SELECT pg_typeof(true::INTEGER);   -- Hasil: 'integer'
```

Dari hasil tersebut dapat dilihat bahwa:

- `true::BOOLEAN` tetap bertipe `boolean`
- `true::INTEGER` sudah benar-benar berubah menjadi `integer`

Ini menegaskan bahwa casting di PostgreSQL tidak hanya mengubah nilai secara konseptual, tetapi juga mengubah **tipe datanya secara eksplisit**.

Dengan memahami cara konversi Boolean ke tipe data lain, Anda dapat menulis query yang lebih fleksibel dan ekspresif, tanpa kehilangan kejelasan tentang makna data yang sedang diproses.

## 3. Hubungan Antar Konsep

**Alur pemahaman:**

1. **Tipe Data Native** → PostgreSQL memiliki implementasi Boolean yang sesungguhnya, bukan hanya wrapper integer
2. **Fleksibilitas Input** → Berbagai format input (true/false, 1/0, yes/no, on/off) membuat interaksi lebih mudah
3. **Automatic Coercion** → Semua format input otomatis dikonversi ke representasi Boolean internal
4. **Efisiensi Storage** → Ukuran 1 byte membuat Boolean sangat efisien untuk menyimpan data ya/tidak
5. **Type Safety** → Meskipun fleksibel, PostgreSQL tetap menjaga keamanan tipe (tidak bisa insert integer langsung tanpa cast)

**Filosofi design:**
PostgreSQL mengutamakan **ketepatan tipe data** untuk merepresentasikan domain data. Boolean untuk true/false, bukan integer yang di-abuse.

## 4. Kesimpulan

PostgreSQL memiliki implementasi Boolean yang **native dan efisien**:

- ✅ Tipe data sesungguhnya (bukan alias integer)
- ✅ Hanya **1 byte** storage (paling efisien)
- ✅ Menerima banyak format input (true/false, 1/0, yes/no, on/off, t/f)
- ✅ Automatic conversion ke representasi Boolean internal
- ✅ Mendukung three-valued logic (true, false, null)

**Prinsip utama:** Gunakan tipe data yang **paling tepat** merepresentasikan data Anda. Untuk data true/false, gunakan Boolean—bukan integer yang di-abuse. Ini membuat kode lebih jelas, lebih type-safe, dan lebih efisien dalam storage.
