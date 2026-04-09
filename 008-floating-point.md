# 📝 Tipe Data Floating-Point di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tipe data floating-point di PostgreSQL, yaitu `REAL` dan `DOUBLE PRECISION`, yang merupakan tipe data untuk bilangan pecahan dengan **presisi variabel dan tidak exact (approximate)**. Berbeda dengan NUMERIC yang lambat tapi perfect, floating-point sangat cepat tetapi menghasilkan aproksimasi. Video mendemonstrasikan perbedaan performa dan presisi antara floating-point dan NUMERIC melalui benchmarking langsung.

## 2. Konsep Utama

### a. Dua Tipe Floating-Point di PostgreSQL

Di PostgreSQL, tipe data floating-point dibagi menjadi dua pilihan utama: **REAL** dan **DOUBLE PRECISION**. Keduanya sama-sama menyimpan angka dalam bentuk _approximation_, tetapi berbeda dalam ukuran memori, jangkauan nilai, dan tingkat presisi. Memahami perbedaannya penting agar kita bisa memilih tipe yang tepat sesuai kebutuhan data dan performa.

#### 1. REAL (4 bytes)

Tipe **REAL** adalah floating-point dengan ukuran paling kecil yang disediakan PostgreSQL.

Secara teknis, REAL memiliki karakteristik berikut:

- Menggunakan **4 bytes** memori
- Memiliki **range** sekitar `1E-37` sampai `1E+37`, yang berarti dapat menyimpan nilai dari 10⁻³⁷ hingga 10³⁷
- Menyediakan **presisi minimal 6 digit** di belakang koma
- Memiliki alias **FLOAT4**

Range ini terlihat sangat besar, sehingga REAL mampu menangani nilai yang sangat kecil maupun sangat besar. Namun, karena hanya menyediakan sekitar 6 digit presisi, detail angka setelah itu bisa hilang atau dibulatkan.

Contoh penggunaannya bisa dilihat pada tabel berikut:

```sql
-- Deklarasi dengan REAL
CREATE TABLE sensor_readings (
    sensor_name TEXT,
    reading REAL
);

-- Contoh values
INSERT INTO sensor_readings VALUES
    ('Temperature', 23.456789),
    ('Pressure', 1013.25),
    ('Humidity', 65.8);

SELECT * FROM sensor_readings;
-- Reading akan disimpan sebagai approximation
```

Ketika data dimasukkan, PostgreSQL tidak menyimpan angka secara persis seperti yang diketik, melainkan sebagai pendekatan biner. Untuk banyak kasus sensor, perbedaan kecil ini tidak menjadi masalah karena nilai tersebut memang berasal dari pengukuran fisik yang tidak sepenuhnya presisi.

REAL biasanya digunakan ketika:

- Menyimpan **sensor data** seperti suhu, tekanan, atau kelembapan
- Menangani **koordinat grafis**
- Melakukan **perhitungan ilmiah sederhana** yang tidak menuntut presisi absolut
- Prioritas utama adalah **kecepatan dan range**, bukan ketelitian sempurna

#### 2. DOUBLE PRECISION (8 bytes)

**DOUBLE PRECISION** adalah versi floating-point yang lebih besar dan lebih presisi. Secara umum, inilah pilihan default ketika kita membutuhkan floating-point yang “aman”.

Karakteristik utamanya:

- Menggunakan **8 bytes** memori (dua kali lipat REAL)
- Memiliki **range sangat luas**, dari `1E-307` hingga `1E+308`
- Menyediakan **presisi minimal 15 digit** di belakang koma
- Memiliki alias **FLOAT8** dan **DOUBLE**

Range dan presisi ini membuat DOUBLE PRECISION mampu menangani perhitungan numerik yang jauh lebih kompleks dan detail dibandingkan REAL.

Contoh penggunaannya:

```sql
-- Deklarasi dengan DOUBLE PRECISION
CREATE TABLE iot_data (
    sensor_name TEXT,
    reading DOUBLE PRECISION
);

-- Atau menggunakan alias
CREATE TABLE measurements (
    value FLOAT8  -- Sama dengan DOUBLE PRECISION
);

-- Contoh values dengan presisi lebih tinggi
INSERT INTO measurements VALUES
    (3.141592653589793),  -- Pi dengan banyak digit
    (2.718281828459045),  -- Euler's number
    (1.414213562373095);  -- √2
```

Pada contoh di atas, PostgreSQL mampu menyimpan lebih banyak digit desimal dibandingkan REAL. Walaupun tetap berupa approximation, hasilnya jauh lebih mendekati nilai matematis sebenarnya.

DOUBLE PRECISION sangat cocok digunakan ketika:

- Membutuhkan **floating-point default** dan tidak ingin terlalu banyak berpikir soal trade-off
- Melakukan **perhitungan ilmiah** dengan presisi lebih tinggi
- Menyimpan **fitur machine learning**
- Mengelola **koordinat geografis** dengan ketelitian tinggi
- Membutuhkan **range dan presisi** yang lebih besar dibandingkan REAL

#### Perbandingan REAL dan DOUBLE PRECISION

Jika diringkas, perbedaannya bisa dilihat dari tiga aspek utama: ukuran, range, dan presisi.

- **REAL (FLOAT4)**
  Menggunakan 4 bytes, memiliki range besar, tetapi presisi hanya sekitar 6 digit.
- **DOUBLE PRECISION (FLOAT8)**
  Menggunakan 8 bytes, memiliki range jauh lebih besar, dan presisi sekitar 15 digit.

Pilihan antara keduanya bukan soal benar atau salah, melainkan soal kebutuhan. Jika data Anda bersifat kasar dan sangat sensitif terhadap performa serta ukuran memori, REAL sudah cukup. Namun jika ragu, atau jika presisi lebih penting daripada penghematan memori, **DOUBLE PRECISION hampir selalu menjadi pilihan yang lebih aman**.

### b. Aliases untuk Floating-Point

Di PostgreSQL, tipe data floating-point memiliki beberapa **alias** atau nama alternatif. Secara fungsional, alias-alias ini merujuk pada tipe data yang sama di level internal. Karena itu, memahami alias ini penting agar kita tidak salah paham saat membaca skema database orang lain, atau saat menulis definisi tabel sendiri.

#### Alias untuk REAL (4 bytes)

Tipe **REAL** memiliki beberapa bentuk penulisan yang semuanya dipetakan ke tipe yang sama, yaitu floating-point 4 bytes.

Perhatikan contoh berikut:

```sql
-- Semua deklarasi ini IDENTIK
CREATE TABLE test1 (value REAL);
CREATE TABLE test2 (value FLOAT4);
CREATE TABLE test3 (value FLOAT(1));   -- FLOAT(1-24) → REAL
CREATE TABLE test4 (value FLOAT(24));  -- Max untuk REAL
```

Meskipun sintaksnya berbeda, PostgreSQL akan menyimpan kolom `value` di keempat tabel tersebut sebagai tipe **real**. Hal ini bisa dibuktikan dengan perintah deskripsi tabel:

```sql
\d test1  -- Type: real
\d test2  -- Type: real
\d test3  -- Type: real
```

Artinya, penggunaan `REAL`, `FLOAT4`, atau `FLOAT(n)` dengan nilai `n` antara 1 sampai 24 tidak menghasilkan perbedaan apa pun pada tipe data yang digunakan. Semuanya berujung pada tipe **REAL** dengan ukuran 4 bytes.

#### Alias untuk DOUBLE PRECISION (8 bytes)

Hal yang sama juga berlaku untuk **DOUBLE PRECISION**, yaitu floating-point 8 bytes. PostgreSQL menyediakan beberapa alias yang semuanya mengarah ke tipe ini.

Contohnya:

```sql
-- Semua deklarasi ini IDENTIK
CREATE TABLE test1 (value DOUBLE PRECISION);
CREATE TABLE test2 (value FLOAT8);
CREATE TABLE test3 (value FLOAT(25));  -- FLOAT(25-53) → DOUBLE PRECISION
CREATE TABLE test4 (value FLOAT(49));
CREATE TABLE test5 (value FLOAT(53));  -- Max untuk DOUBLE PRECISION
```

Jika kita cek struktur tabelnya:

```sql
\d test1  -- Type: double precision
\d test2  -- Type: double precision
```

Hasilnya konsisten: semua kolom tersebut bertipe **double precision**, terlepas dari alias yang digunakan.

#### Memahami Sintaks FLOAT(n)

Secara sintaks, PostgreSQL memang menerima penulisan `FLOAT(n)`. Namun, penting untuk memahami bagaimana sebenarnya PostgreSQL memperlakukannya.

Aturannya adalah:

- `FLOAT(1–24)` akan dikonversi menjadi **REAL**
- `FLOAT(25–53)` akan dikonversi menjadi **DOUBLE PRECISION**

Yang sering mengejutkan banyak orang adalah kenyataan bahwa **nilai `n` itu sendiri sebenarnya diabaikan**. PostgreSQL hanya melihat apakah `n` berada di rentang REAL atau DOUBLE PRECISION, lalu memilih salah satunya. Tidak ada pengaruh lanjutan terhadap presisi aktual di dalam penyimpanan data.

Perilaku ini mirip dengan MySQL, yang juga tidak benar-benar menggunakan parameter `n` untuk mengatur presisi floating-point.

#### Best Practice dalam Penggunaan Alias

Agar skema database mudah dibaca dan tidak menimbulkan kebingungan, ada beberapa praktik yang sebaiknya diikuti.

Contoh yang direkomendasikan:

```sql
-- Direkomendasikan: nama tipe jelas dan eksplisit
CREATE TABLE sensors (
    temp REAL,                   -- Jelas dan standar
    pressure DOUBLE PRECISION     -- Jelas dan standar
);
```

Pendekatan ini membuat siapa pun yang membaca skema langsung memahami ukuran dan karakteristik data yang disimpan.

Alternatif yang masih bisa diterima:

```sql
-- Masih acceptable: eksplisit ukuran dengan FLOAT4/FLOAT8
CREATE TABLE metrics (
    small_value FLOAT4,   -- 4 bytes, jelas
    large_value FLOAT8    -- 8 bytes, jelas
);
```

Penggunaan `FLOAT4` dan `FLOAT8` menekankan ukuran memori secara eksplisit, yang kadang berguna dalam konteks optimasi.

Yang sebaiknya dihindari:

```sql
-- Membingungkan: FLOAT(n)
CREATE TABLE confusing (
    value FLOAT(30)  -- Terlihat "spesial", padahal sama dengan DOUBLE PRECISION
);
```

Penulisan seperti ini berpotensi menyesatkan pembaca skema, karena terlihat seolah-olah `30` memiliki makna khusus, padahal PostgreSQL tetap menyimpannya sebagai **DOUBLE PRECISION**.

Kesimpulannya, **gunakan nama tipe yang jelas dan eksplisit**. Ini bukan hanya soal teknis, tetapi juga soal keterbacaan dan pemeliharaan jangka panjang dari skema database Anda.

### c. Demonstrasi: Imprecision dalam Floating-Point

Salah satu hal terpenting yang perlu dipahami saat bekerja dengan floating-point adalah bahwa **ketidaktepatan (imprecision) bukanlah bug**, melainkan sifat dasar dari cara komputer merepresentasikan angka. Jika kita memahami akar masalahnya, perilaku yang terlihat “aneh” justru akan terasa masuk akal.

#### Problem Fundamental Floating-Point

Floating-point menyimpan angka menggunakan **representasi biner (base-2)**, bukan desimal (base-10) seperti yang biasa kita gunakan sehari-hari. Masalahnya, tidak semua angka desimal bisa direpresentasikan secara **exact** dalam bentuk biner.

Akibatnya, beberapa nilai harus disimpan sebagai pendekatan terdekat, bukan nilai yang benar-benar persis.

#### Contoh Imprecision Sederhana

Perhatikan operasi matematika berikut:

```sql
-- Operasi matematis sederhana
SELECT 7.0::FLOAT8 * 2.0 / 10.0;
-- Expected: 1.4
-- Actual:   1.4000000000000001
```

Secara logika matematika, hasilnya jelas `1.4`. Namun PostgreSQL menampilkan `1.4000000000000001`. Ini **bukan kesalahan PostgreSQL**, dan juga bukan kesalahan perhitungan. Ini adalah konsekuensi dari penggunaan floating-point.

#### Mengapa Hal Ini Terjadi?

Jika kita lihat lebih dalam, angka desimal `1.4` tidak memiliki representasi biner yang berhingga:

```
Decimal: 1.4
Binary:  1.0110011001100110011... (berulang tanpa akhir)
```

Komputer tidak bisa menyimpan deret biner tak berhingga, sehingga harus **memotong (cut off)** di titik tertentu. Pemotongan inilah yang menyebabkan sedikit ketidaktepatan. Selisihnya sangat kecil, tetapi cukup untuk membuat hasil akhirnya bukan `1.4` persis, melainkan `1.4000000000000001`.

#### Masalah Saat Melakukan Perbandingan

Imprecision ini menjadi masalah serius ketika kita melakukan **perbandingan langsung (strict equality)**.

```sql
-- ❌ Strict equality TIDAK AKAN WORK
SELECT (7.0::FLOAT8 * 2.0 / 10.0) = 1.4;
-- Result: false
```

Hasilnya `false` karena komputer membandingkan:

- `1.4000000000000001`
- dengan `1.4`

Keduanya tidak persis sama, meskipun secara matematis kita menganggapnya setara.

#### Solusi: Epsilon Comparison

Cara yang benar untuk membandingkan floating-point adalah dengan **epsilon comparison**, yaitu membandingkan selisih dua angka dan memastikan selisih tersebut cukup kecil.

```sql
-- ✅ Epsilon comparison
SELECT ABS((7.0::FLOAT8 * 2.0 / 10.0) - 1.4) < 0.001;
-- Result: true
```

Logikanya sederhana:
“Jika perbedaan antara dua nilai **lebih kecil dari ambang batas tertentu**, maka kita anggap nilainya sama.”

Nilai ambang batas ini disebut **epsilon**, dan besarnya disesuaikan dengan konteks data.

#### Best Practice untuk Comparison Floating-Point

Dalam query sehari-hari, ada beberapa pola yang sebaiknya diikuti:

```sql
-- Jangan: strict equality
WHERE float_column = 1.4  -- ❌ Berisiko

-- Lakukan: epsilon comparison
WHERE ABS(float_column - 1.4) < 0.0001  -- ✅ Aman

-- Alternatif: gunakan range
WHERE float_column BETWEEN 1.3999 AND 1.4001  -- ✅ Aman
```

Pendekatan ini membuat query lebih robust dan tidak mudah gagal hanya karena perbedaan kecil di digit terakhir.

#### Kontras dengan NUMERIC

Sebagai pembanding, mari lihat bagaimana **NUMERIC** berperilaku:

```sql
-- NUMERIC: perfectly precise
SELECT 7.0::NUMERIC * 2.0 / 10.0;
-- Result: 1.4 (exactly!)

SELECT (7.0::NUMERIC * 2.0 / 10.0) = 1.4;
-- Result: true
```

NUMERIC menggunakan representasi desimal dengan presisi tetap, sehingga hasilnya **benar-benar tepat**. Tidak ada approximation, tidak ada kejutan saat perbandingan.

Inilah alasan utama mengapa **NUMERIC wajib digunakan untuk data finansial**, seperti harga, saldo, dan transaksi. Floating-point sangat cepat dan cocok untuk sains serta sensor, tetapi ketika ketepatan absolut dibutuhkan, NUMERIC adalah pilihan yang tidak bisa ditawar.

### d. Benchmarking: Speed Comparison

Untuk benar-benar memahami perbedaan antara **FLOAT8** dan **NUMERIC**, teori saja tidak cukup. Karena itu, bagian ini mendemonstrasikan sebuah **benchmark nyata** yang mengukur performa kedua tipe data tersebut pada skala besar, yaitu **20 juta baris data**. Tujuannya bukan mencari angka performa yang absolut, tetapi memperlihatkan **perbedaan karakteristik kecepatan** yang nyata dalam kondisi mendekati dunia nyata.

#### Setup Benchmark

Benchmark dilakukan dengan pendekatan yang sangat sederhana namun efektif. PostgreSQL digunakan untuk menghasilkan 20 juta angka menggunakan `generate_series()`, lalu pada setiap baris dilakukan operasi pembagian.

Test pertama menggunakan **FLOAT8**:

```sql
SELECT num::FLOAT8 / 3.0::FLOAT8 AS float_result
FROM generate_series(1, 20000000) AS num;
-- Execution time: 2.3 seconds
```

Pada query ini, setiap nilai dikonversi ke FLOAT8 dan dibagi dengan `3.0`. Operasi ini murni aritmatika floating-point, yang sebagian besar ditangani langsung oleh hardware CPU.

Test kedua menggunakan **NUMERIC**:

```sql
SELECT num::NUMERIC / 3.0::NUMERIC AS numeric_result
FROM generate_series(1, 20000000) AS num;
-- Execution time: 4.5–4.6 seconds
```

Di sini, operasi yang sama dilakukan, tetapi menggunakan tipe NUMERIC. Karena NUMERIC mengelola presisi secara software, setiap operasi membutuhkan lebih banyak instruksi dibandingkan FLOAT8.

#### Hasil Benchmark

Hasilnya cukup jelas dan konsisten:

- **FLOAT8** menyelesaikan operasi dalam sekitar **2.3 detik**
- **NUMERIC** membutuhkan sekitar **4.5 hingga 4.6 detik**

Secara kasar, FLOAT8 hampir **dua kali lebih cepat** dibandingkan NUMERIC. Atau dilihat dari sisi lain, NUMERIC sekitar **dua kali lebih lambat** dibandingkan FLOAT8.

#### Interpretasi Hasil

Dari benchmark ini, beberapa kesimpulan penting bisa diambil:

- **FLOAT8 memang secara signifikan lebih cepat** daripada NUMERIC
- Perbedaan ini semakin terasa pada **operasi matematis yang intensif**
- Pada skala 20 juta operasi, selisih sekitar **2 detik** sudah sangat mencolok
- Dalam aplikasi real-time, data analytics, atau sistem ber-throughput tinggi, selisih ini bisa menjadi faktor penentu

Benchmark ini menunjukkan bahwa pilihan tipe data bukan sekadar soal akurasi, tetapi juga soal **kinerja sistem secara keseluruhan**.

#### Catatan Penting tentang Benchmark

Benchmark ini memang tidak dirancang sebagai eksperimen ilmiah yang sempurna. Seperti yang dicatat dalam video:

> “Not a very scientific benchmark, but hopefully it's big enough that we'll see a pretty big difference.”

Namun justru karena kesederhanaannya, benchmark ini efektif untuk **mendemonstrasikan trade-off utama** antara kedua tipe data:

- **NUMERIC**: lambat, tetapi akurat secara sempurna
- **FLOAT**: sangat cepat, tetapi menggunakan approximation

Intinya, jika aplikasi Anda memprioritaskan **kecepatan dan skala**, FLOAT8 sering menjadi pilihan yang lebih tepat. Sebaliknya, jika **ketepatan absolut** adalah syarat utama, maka NUMERIC tetap tidak tergantikan, meskipun harus membayar harga dari sisi performa.

### e. Fungsi generate_series(): Bonus Tool

Selain tipe data dan performa, PostgreSQL juga menyediakan beberapa fungsi bawaan yang sangat membantu saat eksplorasi, testing, dan benchmarking. Salah satu yang paling berguna adalah **`generate_series()`**. Fungsi ini sering dianggap “bonus tool” karena sederhana, tetapi kekuatannya luar biasa.

#### Pengenalan Singkat

`generate_series()` digunakan untuk menghasilkan deretan nilai secara otomatis, tanpa perlu membuat tabel atau melakukan `INSERT`.

Contoh paling dasar:

```sql
-- Generate series angka dari 1 sampai 20
SELECT * FROM generate_series(1, 20);
```

Hasilnya adalah satu kolom dengan nilai berurutan:

```
1
2
3
...
20
```

PostgreSQL akan menghasilkan baris demi baris sesuai rentang yang kita minta, seolah-olah data tersebut memang sudah tersimpan di tabel.

#### Kegunaan generate_series()

Fungsi ini sangat fleksibel dan sering dipakai dalam berbagai skenario praktis, antara lain:

- **Generate test data secara instan**, tanpa perlu repot membuat tabel dan menulis banyak perintah `INSERT`
- **Benchmarking performa**, terutama untuk menguji operasi matematis atau query kompleks
- **Membuat sequence sementara** untuk testing logika aplikasi
- **Populate data sementara**, misalnya untuk join atau simulasi data

Karena data dihasilkan secara dinamis, `generate_series()` sangat cocok untuk eksperimen dan eksplorasi.

#### Sintaks Lengkap dan Contoh

Bentuk paling sederhana dari `generate_series()` adalah menentukan nilai awal dan akhir:

```sql
-- Basic: start sampai end
generate_series(start, end)
```

Kita juga bisa menentukan langkah (step):

```sql
-- Dengan step
generate_series(start, end, step)
```

Contoh penggunaannya:

```sql
SELECT * FROM generate_series(1, 10, 2);
-- Result: 1, 3, 5, 7, 9
```

Di sini PostgreSQL menghasilkan angka dari 1 sampai 10 dengan lompatan 2.

Selain angka, `generate_series()` juga mendukung **tanggal dan interval**, yang sangat berguna untuk kebutuhan reporting berbasis waktu:

```sql
SELECT * FROM generate_series(
    '2024-01-01'::DATE,
    '2024-01-10'::DATE,
    '1 day'::INTERVAL
);
```

Query ini akan menghasilkan deretan tanggal selama 10 hari berturut-turut, dari 1 Januari sampai 10 Januari 2024.

#### Use Case dalam Benchmark

Dalam konteks benchmarking, `generate_series()` benar-benar bersinar. Kita bisa menghasilkan jutaan baris data tanpa menyimpan apa pun ke disk.

Contohnya:

```sql
SELECT
    num::FLOAT8 / 3.0::FLOAT8 AS float_result
FROM generate_series(1, 20000000) AS num;
```

Query ini menghasilkan **20 juta baris** angka, lalu langsung melakukan operasi matematis pada setiap baris. Tidak ada tabel permanen, tidak ada `INSERT`, dan tidak ada overhead penyimpanan.

Inilah alasan mengapa `generate_series()` sering digunakan untuk **performance testing** dan eksperimen skala besar. Sederhana, cepat, dan sangat powerful—alat kecil dengan dampak besar dalam workflow PostgreSQL sehari-hari.

### f. Decision Matrix: Kapan Gunakan Apa?

**Flowchart pemilihan tipe:**

```
Apakah data berupa angka?
├─ Tidak → TEXT/VARCHAR
└─ Ya → Lanjut

Apakah butuh fractional numbers?
├─ Tidak → INTEGER (SMALLINT/INT/BIGINT)
└─ Ya → Lanjut

Apakah ini FINANCIAL data (uang)?
├─ Ya → NUMERIC | INTEGER
└─ Tidak → Lanjut

Apakah akurasi sempurna MUTLAK diperlukan?
├─ Ya → NUMERIC
└─ Tidak (approximation OK) → Lanjut

Apakah butuh presisi > 6 digits?
├─ Ya → DOUBLE PRECISION (15 digits)
└─ Tidak → REAL (6 digits) atau DOUBLE PRECISION
```

**Tabel decision:**

| Use Case                      | Recommended Type   | Reason                          |
| ----------------------------- | ------------------ | ------------------------------- |
| **Harga, uang, salary**       | `NUMERIC(10,2)`    | Must be exact                   |
| **Interest rates**            | `NUMERIC(5,4)`     | Financial precision             |
| **Temperature sensor**        | `REAL`             | Approximation OK, speed matters |
| **GPS coordinates**           | `DOUBLE PRECISION` | Need high precision             |
| **Scientific calculations**   | `DOUBLE PRECISION` | Range + precision               |
| **Graphics (x, y, z)**        | `REAL` or `FLOAT8` | Speed critical                  |
| **Machine learning features** | `DOUBLE PRECISION` | Common standard                 |
| **Counter, quantity**         | `INTEGER`          | Whole numbers only              |

**Real-world examples:**

```sql
-- IoT sensor readings (approximation OK)
CREATE TABLE sensor_data (
    sensor_id INTEGER,
    temperature REAL,           -- ✅ Speed > precision
    humidity REAL,              -- ✅ Approximation OK
    timestamp TIMESTAMP
);

-- E-commerce transactions (must be exact)
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    subtotal NUMERIC(10, 2),    -- ✅ Must be exact
    tax NUMERIC(10, 2),         -- ✅ Must be exact
    total NUMERIC(10, 2)        -- ✅ Must be exact
);

-- Scientific research (high precision needed)
CREATE TABLE experiments (
    experiment_id INTEGER,
    measurement DOUBLE PRECISION, -- ✅ High precision
    error_margin DOUBLE PRECISION -- ✅ High precision
);

-- 3D graphics coordinates
CREATE TABLE vertices (
    vertex_id INTEGER,
    x REAL,                     -- ✅ Speed for rendering
    y REAL,                     -- ✅ Speed for rendering
    z REAL                      -- ✅ Speed for rendering
);
```

## 3. Hubungan Antar Konsep

**Evolution pemahaman tipe data numerik:**

```
Video 1: INTEGER
├─ Fast, accurate, whole numbers only
├─ SMALLINT (2 bytes), INTEGER (4 bytes), BIGINT (8 bytes)
└─ Foundation: Speed + accuracy

Video 2: NUMERIC
├─ Slow, accurate, support fractional
└─ Trade-off: Accuracy vs Speed

Video 3: FLOATING-POINT (sekarang)
├─ Fast, approximate, support fractional
├─ Best of both worlds (speed + fractional)
└─ Trade-off: Speed vs Accuracy

Kesimpulan: "Pick the one that works for you"
```

**Relationship map:**

```
          Speed
            ↑
            |
    INTEGER |     FLOAT
       ⚡⚡⚡|⚡⚡⚡
            |
    --------|--------→ Support Fractions
            |
      NUMERIC|
         🐌 |
            |
            ↓
         Accuracy
```

**Key principle dari seluruh series:**

> **"Perfectly accurate, perfectly accurate approximation, really fast, kind of slow, really fast."**

Artinya:

- INTEGER: Perfectly accurate, really fast (tapi whole only)
- NUMERIC: Perfectly accurate, kind of slow (tapi fractional)
- FLOAT: Approximation, really fast (fractional)

**Trade-off triangle:**

```
         SPEED
         /   \
        /     \
    INTEGER   FLOAT
       |       |
       |       |
    NUMERIC ← ACCURACY → (varies)
```

## 4. Kesimpulan

Floating-point numbers (`REAL` dan `DOUBLE PRECISION`) melengkapi spektrum tipe data numerik di PostgreSQL dengan memberikan kombinasi **kecepatan tinggi dan support fractional numbers**, dengan trade-off berupa **presisi yang approximate (inexact)**.

**Key takeaways:**

1. **Dua tipe utama:**

   - `REAL` (4 bytes): Range 1E-37 to 1E+37, precision ≥6 digits
   - `DOUBLE PRECISION` (8 bytes): Range 1E-307 to 1E+308, precision ≥15 digits

2. **Karakteristik floating-point:**

   - ⚡ **Sangat cepat** (~2x lebih cepat dari NUMERIC)
   - ❌ **Inexact/approximate** (bisa ada precision loss)
   - ✅ **Support fractional** numbers
   - ⚠️ **JANGAN untuk financial data!**

3. **Aliases tersedia:**

   - `FLOAT4` = `REAL`
   - `FLOAT8` = `DOUBLE PRECISION`
   - `FLOAT(1-24)` → `REAL`
   - `FLOAT(25-53)` → `DOUBLE PRECISION`

4. **Imprecision reality:**

   - `7.0 * 2.0 / 10.0` = `1.4000000000000001` (bukan exactly 1.4)
   - Gunakan **epsilon comparison**, bukan strict equality
   - Ini bukan bug - ini fundamental nature of binary floating-point

5. **Performance matters:**

   - Benchmark nyata: FLOAT8 (2.3s) vs NUMERIC (4.5s) untuk 20M operations
   - Perbedaan signifikan untuk aplikasi real-time atau data-intensive

6. **Use cases:**
   - ✅ FLOAT: IoT sensors, scientific calculations, graphics, ML features
   - ❌ FLOAT: Money, accounting, financial data (use NUMERIC!)

**Final wisdom:**

> "Pick the one that works for you. Hopefully we have proven that all of those underlying fundamental assumptions are accurate. And now you are well equipped to pick the one that matches your data and your use case."

**Complete spectrum:**

- **INTEGER:** Perfect accuracy + super fast = untuk whole numbers
- **NUMERIC:** Perfect accuracy + slow = untuk money (WAJIB!)
- **FLOAT:** Approximate + super fast = untuk science, IoT, graphics

Trade-off adalah real - tidak ada "one size fits all". Pilih berdasarkan requirements spesifik aplikasi Anda: accuracy vs speed, fractional vs whole, range yang dibutuhkan.
