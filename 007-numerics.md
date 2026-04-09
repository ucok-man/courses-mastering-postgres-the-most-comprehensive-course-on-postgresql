# 📝 Tipe Data Numeric (DECIMAL) di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tipe data `NUMERIC` (alias `DECIMAL`) di PostgreSQL, yang merupakan tipe data untuk bilangan pecahan dengan presisi sempurna. Pembahasan mencakup perbandingan antara INTEGER, NUMERIC, dan floating-point, serta cara menggunakan parameter precision dan scale untuk mengontrol format angka. Tipe ini ideal untuk data finansial yang membutuhkan akurasi mutlak.

## 2. Konsep Utama

### a. Spektrum Tipe Data Numerik

Dalam PostgreSQL, angka tidak disimpan dengan satu jenis tipe data saja. Ada **tiga kategori utama** tipe data numerik, dan masing-masing berada pada posisi yang berbeda dalam spektrum **kecepatan, akurasi, dan presisi**. Memahami spektrum ini penting agar kita bisa memilih tipe data yang paling tepat sesuai kebutuhan aplikasi.

#### Gambaran Umum Spektrum Tipe Data

Secara konseptual, ketiga kategori ini bisa dibayangkan berada pada satu garis lurus:

```
[INTEGER] ←→ [NUMERIC/DECIMAL] ←→ [FLOATING-POINT]
  Fast         Slow, Precise       Fast
  Accurate     Accurate            Approximate
  Whole only   Fractional          Fractional
```

Posisi ini bukan tanpa alasan. Di satu sisi ada tipe yang **sangat cepat dan sederhana**, di sisi lain ada tipe yang **sangat fleksibel tetapi mahal secara komputasi**, dan di ujung lainnya ada tipe yang **cepat namun mengorbankan ketepatan absolut**.

#### Perbandingan Karakteristik Utama

Jika kita bedah lebih detail, perbedaan ketiganya terlihat jelas pada aspek berikut:

| Aspek               | INTEGER                 | NUMERIC / DECIMAL          | FLOATING-POINT           |
| ------------------- | ----------------------- | -------------------------- | ------------------------ |
| Support fractional  | ❌ Hanya bilangan bulat | ✅ Ya                      | ✅ Ya                    |
| Akurasi nilai       | ✅ Sempurna             | ✅ Sempurna                | ❌ Aproksimasi           |
| Presisi perhitungan | ✅ Tidak ada loss       | ✅ Tidak ada loss          | ❌ Bisa hilang presisi   |
| Kecepatan operasi   | ⚡ Sangat cepat         | 🐌 Paling lambat           | ⚡ Sangat cepat          |
| Use case umum       | Counter, ID, quantity   | Uang, finansial, akuntansi | Sains, grafik, statistik |

Dari tabel ini sudah terlihat bahwa **tidak ada tipe data yang “paling benar” secara universal**. Semuanya bergantung pada konteks masalah yang ingin diselesaikan.

#### INTEGER: Cepat dan Akurat untuk Bilangan Bulat

Tipe **INTEGER** digunakan ketika nilai yang disimpan **selalu bilangan bulat** dan tidak membutuhkan pecahan.

Contoh penggunaan:

```sql
CREATE TABLE inventory (
  stock INTEGER
);

INSERT INTO inventory (stock) VALUES (10);
```

Jika kita melakukan operasi aritmatika:

```sql
SELECT stock + 5 AS total_stock FROM inventory;
```

Hasilnya akan selalu **akurat dan cepat**, karena PostgreSQL hanya bekerja dengan bilangan bulat tanpa perlu memikirkan presisi desimal.

INTEGER sangat ideal untuk:

- ID atau primary key
- Counter
- Jumlah item atau unit

Namun, INTEGER **tidak cocok** jika ada nilai pecahan sekecil apa pun.

#### NUMERIC / DECIMAL: Lambat tapi Presisi Mutlak

Tipe **NUMERIC** (atau **DECIMAL**) dirancang untuk kasus di mana **ketepatan nilai adalah prioritas utama**, bahkan jika harus mengorbankan performa.

Contoh penggunaan pada data finansial:

```sql
CREATE TABLE payments (
  amount NUMERIC(10, 2)
);

INSERT INTO payments (amount) VALUES (1250.75);
```

Jika kita menjumlahkan nilai:

```sql
SELECT amount + 0.10 AS total FROM payments;
```

Hasilnya akan **tepat 1250.85**, tanpa kesalahan pembulatan tersembunyi. Inilah alasan utama mengapa NUMERIC selalu direkomendasikan untuk:

- Uang
- Perhitungan pajak
- Akuntansi dan laporan keuangan

Trade-off-nya jelas: operasi NUMERIC lebih lambat karena PostgreSQL harus menjaga presisi digit demi digit.

#### FLOATING-POINT: Cepat tapi Bersifat Aproksimasi

Tipe **FLOATING-POINT** (REAL atau DOUBLE PRECISION) digunakan untuk perhitungan yang membutuhkan **pecahan** dan **kecepatan tinggi**, tetapi masih bisa mentoleransi kesalahan kecil.

Contoh:

```sql
SELECT 0.1 + 0.2;
```

Hasilnya mungkin terlihat seperti:

```
0.30000000000000004
```

Ini bukan bug, melainkan konsekuensi dari cara bilangan desimal direpresentasikan secara biner. Oleh karena itu, floating-point cocok untuk:

- Perhitungan ilmiah
- Grafik
- Statistik
- Data sensor

Tetapi **tidak boleh digunakan** untuk nilai yang menuntut presisi absolut seperti uang.

#### Analogi Sederhana untuk Mempermudah Pemahaman

Agar lebih intuitif, bayangkan ketiga tipe data ini seperti berikut:

```
INTEGER        = Menghitung kelereng (1, 2, 3, 4...)
NUMERIC        = Menghitung uang (Rp 1.250,75) — harus tepat
FLOATING-POINT = Mengukur suhu (23.456°C) — sedikit meleset masih wajar
```

Dengan analogi ini, kita bisa melihat bahwa pemilihan tipe data bukan sekadar masalah teknis, tetapi juga soal **memahami sifat data yang kita kelola**.

### b. NUMERIC vs DECIMAL: Sama Persis

Dalam PostgreSQL, sering muncul pertanyaan apakah `NUMERIC` dan `DECIMAL` itu berbeda, atau mana yang “lebih benar” untuk digunakan. Jawabannya cukup tegas: **keduanya sama persis**. Tidak ada perbedaan perilaku, performa, maupun tingkat presisi di antara keduanya.

#### Fakta Penting tentang NUMERIC dan DECIMAL

Beberapa poin kunci yang perlu dipahami sejak awal:

- `NUMERIC` dan `DECIMAL` adalah **tipe data yang identik** di PostgreSQL
- `DECIMAL` hanyalah **alias** dari `NUMERIC`
- Keduanya bisa digunakan **secara bergantian** tanpa dampak apa pun pada hasil perhitungan atau penyimpanan data

Artinya, PostgreSQL tidak memperlakukan keduanya sebagai dua tipe yang berbeda. Di level internal database, mereka direpresentasikan dengan cara yang sama.

#### Pembuktian Langsung di PostgreSQL

Kita bisa membuktikan hal ini dengan contoh sederhana. Pertama, buat dua tabel dengan deklarasi tipe data yang berbeda secara penulisan:

```sql
-- Deklarasi dengan DECIMAL
CREATE TABLE test1 (
    interest_rate DECIMAL
);

-- Deklarasi dengan NUMERIC
CREATE TABLE test2 (
    interest_rate NUMERIC
);
```

Sekarang, periksa struktur tabel menggunakan perintah `\d` di `psql`:

```sql
\d test1
-- Column: interest_rate | Type: numeric

\d test2
-- Column: interest_rate | Type: numeric
```

Hasilnya jelas terlihat. Meskipun kita mendefinisikan satu kolom dengan `DECIMAL` dan yang lain dengan `NUMERIC`, PostgreSQL menampilkan **tipe kolom yang sama**, yaitu `numeric`. Ini menegaskan bahwa keduanya memang satu tipe data yang identik.

#### Implikasi Praktis dalam Pengembangan

Karena tidak ada perbedaan teknis sama sekali, pemilihan antara `NUMERIC` dan `DECIMAL` lebih bersifat **gaya penulisan dan konsistensi tim**. Banyak developer dan instruktur memilih `NUMERIC` karena:

- Lebih eksplisit dan langsung mencerminkan nama tipe internal PostgreSQL
- Umum digunakan dalam dokumentasi resmi PostgreSQL

Namun, menggunakan `DECIMAL` juga sepenuhnya valid. Yang terpenting adalah:

- Pilih salah satu
- Gunakan secara **konsisten** di seluruh codebase

Konsistensi ini akan membuat skema database lebih rapi, mudah dibaca, dan menghindari kebingungan yang tidak perlu bagi developer lain di masa depan.

### c. NUMERIC Tanpa Parameter: Unbounded Precision

Secara default, tipe data `NUMERIC` di PostgreSQL **tidak wajib** didefinisikan dengan parameter presisi dan skala. Ketika kita menuliskannya tanpa parameter apa pun, PostgreSQL akan memperlakukannya sebagai **numeric dengan presisi tak terbatas (unbounded precision)**. Ini memberikan fleksibilitas yang sangat besar, tetapi juga membawa konsekuensi yang perlu dipahami dengan baik.

#### Karakteristik NUMERIC Tanpa Parameter

Perhatikan deklarasi tabel berikut:

```sql
CREATE TABLE unlimited (
    value NUMERIC  -- Tanpa parameter apa pun
);
```

Kolom `value` di sini tidak memiliki batasan jumlah digit maupun jumlah angka di belakang koma. Akibatnya, PostgreSQL memberikan beberapa kemampuan penting:

1. **Tidak ada batasan range nilai**
   Nilai yang disimpan bisa jauh lebih besar daripada kapasitas `BIGINT`. Selama memori mencukupi, PostgreSQL akan tetap bisa menyimpannya.

2. **Presisi sempurna**
   Semua digit disimpan apa adanya. Tidak ada pembulatan tersembunyi dan tidak ada kehilangan presisi, bahkan untuk angka yang sangat panjang.

3. **Mendukung bilangan desimal**
   Angka pecahan dengan jumlah digit di belakang koma yang banyak tetap bisa disimpan secara akurat.

4. **Perilakunya mirip kolom TEXT**
   Dari sudut pandang fleksibilitas input, kolom ini “menerima hampir apa saja” selama masih bisa diparse sebagai angka. Tidak ada validasi bawaan terkait panjang atau format angka.

#### Contoh: Angka yang Lebih Besar dari BIGINT

Sebagai pembanding, mari lihat batas maksimum `BIGINT`:

```text
±9,223,372,036,854,775,807
```

Sekarang coba gunakan angka yang jauh lebih besar dari itu.

```sql
-- ❌ Angka terlalu besar untuk BIGINT
SELECT 92233720368547758079223372036854775807::BIGINT;
-- ERROR: bigint out of range
```

PostgreSQL langsung menolak karena nilai tersebut berada di luar jangkauan `BIGINT`.

Namun, dengan `NUMERIC`, hasilnya berbeda:

```sql
-- ✅ NUMERIC bisa menangani angka sebesar ini
SELECT 92233720368547758079223372036854775807::NUMERIC;
-- Result: 92233720368547758079223372036854775807
```

Bahkan jika kita menambahkan pecahan desimal:

```sql
-- ✅ NUMERIC juga mendukung desimal dengan presisi penuh
SELECT 92233720368547758079223372036854775807.12345::NUMERIC;
-- Result: 92233720368547758079223372036854775807.12345
```

Hasilnya tetap persis sama dengan input. Tidak ada pembulatan, tidak ada digit yang hilang.

#### Kapan Sebaiknya Menggunakan NUMERIC Unbounded

Tipe ini sangat berguna dalam kondisi tertentu, misalnya:

- Ketika **batas nilai data belum diketahui** dan bisa berubah di masa depan
- Menyimpan data **finansial berskala global**, dengan berbagai mata uang dan nilai yang sangat besar
- **Perhitungan ilmiah atau matematis** yang membutuhkan presisi arbitrer dan tidak boleh ada kesalahan pembulatan

Dalam kasus-kasus ini, fleksibilitas dan presisi lebih penting daripada performa.

#### Trade-off yang Perlu Dipertimbangkan

Seperti semua desain teknis, ada konsekuensi yang menyertainya:

- **Pro:**
  Fleksibilitas maksimal, tidak ada batasan range atau presisi, dan aman dari kehilangan digit.

- **Con:**
  Operasi sangat lambat dibanding tipe numerik lain, dan tidak ada validasi bawaan terkait panjang atau skala angka.

Karena itu, `NUMERIC` tanpa parameter sebaiknya digunakan **secara sadar dan terkontrol**, hanya ketika fleksibilitas dan presisi absolut benar-benar dibutuhkan.

### d. NUMERIC dengan Parameter: Precision dan Scale

Selain digunakan tanpa parameter, tipe data `NUMERIC` juga bisa didefinisikan dengan aturan yang lebih ketat menggunakan **precision** dan **scale**. Pendekatan ini sangat berguna ketika kita ingin **mengontrol format angka secara eksplisit**, baik dari sisi jumlah digit total maupun jumlah digit di belakang koma.

#### Sintaks Dasar NUMERIC(precision, scale)

Bentuk umum penulisannya adalah:

```sql
NUMERIC(precision, scale)
```

Dua parameter ini menentukan bagaimana PostgreSQL menyimpan dan memvalidasi angka yang masuk.

#### Definisi Precision dan Scale

Agar tidak tertukar, penting memahami arti masing-masing parameter:

- **Precision**
  Menyatakan **total jumlah digit** yang boleh dimiliki sebuah angka, dihitung dari kiri hingga kanan, **tanpa memedulikan posisi titik desimal**.

- **Scale**
  Menyatakan **jumlah digit di sebelah kanan titik desimal**.

Sisa digit yang tersedia (precision − scale) otomatis menjadi jumlah digit maksimum di sebelah kiri titik desimal.

#### Ilustrasi Visual Precision dan Scale

Misalkan kita memiliki angka berikut:

```
12.345
```

Struktur digitnya bisa dijelaskan seperti ini:

```
├─ Precision = 5  (total digit: 1, 2, 3, 4, 5)
└─ Scale     = 3  (digit setelah titik: 3, 4, 5)

Left side : 2 digits (1, 2)
Right side: 3 digits (3, 4, 5)
```

Artinya, jika kita menggunakan `NUMERIC(5, 3)`, maka:

- Maksimal **5 digit total**
- Tepat **3 digit di belakang koma**
- Maksimal **2 digit di depan koma**

#### Contoh Implementasi NUMERIC(5, 3)

Sekarang kita terapkan konsep ini ke dalam tabel PostgreSQL:

```sql
-- NUMERIC(5, 3) = 5 total digit, 3 digit di kanan decimal
CREATE TABLE measurements (
    value NUMERIC(5, 3)
);
```

##### Contoh Nilai yang Valid

```sql
INSERT INTO measurements VALUES (12.345);
```

Hasilnya:

```
12.345
```

Penjelasan:

- Total digit = 5 → valid
- Digit di belakang koma = 3 → valid

Nilai ini sepenuhnya sesuai dengan aturan `(5, 3)`.

##### Contoh Nilai yang Ditolak

```sql
INSERT INTO measurements VALUES (123.45);
```

PostgreSQL akan menghasilkan error:

```
ERROR: numeric field overflow
```

Alasannya:

- Angka `123.45` memang memiliki total 5 digit
- Tetapi digit di sebelah kiri titik desimal ada **3 digit**
- Padahal `NUMERIC(5, 3)` hanya mengizinkan **2 digit di sebelah kiri**

Karena melanggar batas tersebut, data langsung ditolak.

##### Contoh Nilai dengan Rounding Otomatis

```sql
INSERT INTO measurements VALUES (12.3456);
```

Hasil yang tersimpan:

```
12.346
```

Penjelasannya:

- Scale ditetapkan ke 3 digit
- Digit ke-4 setelah koma adalah `6`
- PostgreSQL melakukan **pembulatan (rounding)** ke digit ke-3

Inilah perilaku standar `NUMERIC` dengan parameter: data yang masih masuk dalam batas precision akan **dibulatkan**, bukan ditolak.

#### Kapan Menggunakan NUMERIC dengan Parameter

Pendekatan ini sangat cocok ketika:

- Format angka harus **konsisten**
- Ada aturan bisnis yang jelas, misalnya nilai uang, persentase, atau pengukuran
- Kita ingin **validasi otomatis di level database**, bukan hanya di aplikasi

Dengan `NUMERIC(precision, scale)`, database membantu menjaga kualitas data sejak awal, bukan sekadar menjadi tempat penyimpanan angka.

### e. Parameter NUMERIC: Variasi Penggunaan

Selain bentuk standar `NUMERIC(precision, scale)`, PostgreSQL juga menyediakan beberapa variasi penggunaan parameter yang cukup fleksibel. Variasi ini sering kali membingungkan di awal, tetapi sangat powerful jika dipahami dengan benar, terutama untuk kebutuhan validasi dan pembulatan angka langsung di level database.

#### 1. Single Parameter (Precision Only)

Ketika `NUMERIC` hanya diberikan **satu parameter**, maka parameter tersebut dianggap sebagai **precision**, dan **scale otomatis bernilai 0**.

```sql
-- NUMERIC(5) = precision 5, scale default 0
CREATE TABLE whole_numbers (
    value NUMERIC(5)  -- Sama dengan NUMERIC(5, 0)
);
```

Dengan definisi ini, kolom hanya akan menyimpan **bilangan bulat** hingga 5 digit.

```sql
-- ✅ Valid (bilangan bulat)
INSERT INTO whole_numbers VALUES (12345);
-- Result: 12345
```

Nilai `12345` memiliki 5 digit dan tidak memiliki desimal, sehingga sepenuhnya valid.

```sql
-- ✅ Valid dengan rounding
INSERT INTO whole_numbers VALUES (123.45);
-- Result: 123
```

Pada contoh ini, input memiliki desimal, tetapi karena `scale = 0`, PostgreSQL **menghilangkan bagian desimal dengan pembulatan**. Digit `4` setelah koma menyebabkan nilai dibulatkan ke bawah menjadi `123`.

Jika kita menuliskan definisi secara eksplisit:

```sql
CREATE TABLE explicit (
    value NUMERIC(5, 0)  -- Sama persis dengan NUMERIC(5)
);
```

Hasil dan perilakunya **identik**. Tidak ada perbedaan antara `NUMERIC(5)` dan `NUMERIC(5, 0)`.

#### 2. Negative Scale (Pembulatan ke Kiri Desimal)

Fitur ini tersedia mulai **PostgreSQL 15 ke atas** dan memungkinkan `NUMERIC(p, s)` menggunakan **scale negatif**. Artinya, pembulatan dilakukan **ke kiri titik desimal**, bukan ke kanan seperti kasus scale positif.

```sql
-- NUMERIC(5, -2) = pembulatan 2 digit ke kiri dari decimal point
CREATE TABLE rounded (
    value NUMERIC(5, -2)
);
```

Dengan definisi ini, PostgreSQL akan membulatkan angka ke ratusan terdekat.

##### Contoh 1: Angka Besar tanpa Desimal

```sql
INSERT INTO rounded VALUES (1234567);
-- Result: 1234600
```

Proses yang terjadi di balik layar:

1. PostgreSQL melihat input `1234567` yang memiliki 7 digit signifikan
2. Precision diset ke 5, sehingga hanya **5 digit signifikan** yang boleh dipertahankan sebelum pembulatan
3. Scale = -2 berarti pembulatan dilakukan **2 digit dari kanan**
4. Angka dibulatkan menjadi `12346`, lalu ditambahkan nol padding → `1234600`
5. Jumlah digit signifikan tetap 5 (`12346`), sehingga masih valid

##### Contoh 2: Angka dengan Desimal

```sql
INSERT INTO rounded VALUES (12345.67);
-- Result: 12300
```

Langkahnya serupa:

1. PostgreSQL mengambil 5 digit signifikan dari `12345`
2. Pembulatan dilakukan 2 digit ke kiri
3. Hasilnya menjadi `12300`
4. Digit signifikan tetap 5, sehingga nilai diterima

#### Konsep Penting: Significant Digits

Untuk memahami perilaku ini, kuncinya ada pada **significant digits**.

```
Input       : 1234567
Precision   : 5
Scale       : -2
```

- Digit signifikan adalah digit **sebelum pembulatan dan sebelum nol padding**
- Nol yang ditambahkan akibat scale negatif **tidak dihitung** sebagai digit signifikan

PostgreSQL akan:

1. Mengidentifikasi jumlah digit signifikan awal
2. Melakukan pembulatan sesuai `scale`
3. Memastikan hasil pembulatan masih berada dalam batas `precision`

#### Perhatian: Output Bisa Lebih Panjang dari Precision

Hal penting yang sering mengejutkan:

- **Precision tidak membatasi panjang output secara literal**
- Precision hanya membatasi **jumlah digit signifikan hasil pembulatan**
- Nol hasil padding karena scale negatif **diabaikan dalam perhitungan precision**

Itulah sebabnya nilai seperti `1234600` tetap valid untuk `NUMERIC(5, -2)`.

#### Contoh Kasus Error: Numeric Field Overflow

```sql
SELECT 12345678::NUMERIC(5, -2);
-- ERROR: numeric field overflow
```

Penjelasannya:

```
Input       : 12345678
Precision   : 5
Scale       : -2
```

1. Input memiliki 8 digit signifikan
2. PostgreSQL mencoba membulatkan 2 digit terakhir (`78`)
3. Hasil pembulatan menjadi `12345700`
4. Nilai ini masih memiliki **6 digit signifikan** (`123457`)
5. Melebihi batas precision (5)

Karena hasil pembulatan melanggar aturan precision, PostgreSQL menolak nilai tersebut.

#### Inti Pemahaman

Meskipun `scale` negatif terlihat “hanya menambahkan nol di belakang”, PostgreSQL tetap melakukan validasi ketat terhadap **jumlah digit signifikan hasil pembulatan**. Jika jumlah digit signifikan tersebut melebihi `precision`, maka error `numeric field overflow` akan terjadi.

Memahami detail ini sangat penting ketika menggunakan `NUMERIC` dengan scale negatif, terutama untuk kasus pembulatan massal atau data statistik berskala besar.

### f. Best Practices untuk NUMERIC

Setelah memahami berbagai variasi `NUMERIC`, langkah berikutnya adalah menerapkannya secara **bijak dan konsisten**. Best practices di bawah ini membantu memastikan data tetap akurat, performa terjaga, dan skema database mudah dipahami oleh developer lain.

#### 1. Untuk Data Finansial (Uang)

Untuk semua kasus yang berhubungan dengan uang, **NUMERIC adalah pilihan wajib**. Alasannya sederhana: uang tidak boleh mengalami kesalahan pembulatan sekecil apa pun.

```sql
-- ✅ RECOMMENDED: Always use NUMERIC for money
CREATE TABLE transactions (
    amount NUMERIC(10, 2)  -- Max 99,999,999.99
);
```

Dengan `NUMERIC(10, 2)`:

- Total digit = 10
- Digit desimal = 2
- Nilai maksimum = `99,999,999.99`

Ini sudah sangat cukup untuk:

- Transaksi retail dalam Rupiah (IDR)
- Mayoritas transaksi bisnis dalam Dollar (USD)

```text
IDR: ~100 juta rupiah per transaksi
USD: ~100 juta dollar per transaksi
```

Jika aplikasi Anda menangani transaksi dengan nilai jauh lebih besar, cukup tingkatkan precision-nya:

```sql
CREATE TABLE large_transactions (
    amount NUMERIC(15, 2)  -- Max 9,999,999,999,999.99
);
```

Pendekatan ini tetap aman secara presisi, sekaligus memberi batasan yang jelas pada data.

#### 2. Untuk Interest Rate dan Persentase

Interest rate biasanya membutuhkan:

- Banyak digit di belakang koma
- Nilai yang relatif kecil
- Presisi tinggi dan konsisten

Contoh yang umum adalah menggunakan `NUMERIC(5, 4)`:

```sql
CREATE TABLE loans (
    description TEXT,
    interest_rate NUMERIC(5, 4)  -- Contoh: 5.2500%
);
```

Dengan konfigurasi ini:

- Maksimal 1 digit di depan koma
- 4 digit di belakang koma

Contoh nilai yang valid:

```sql
INSERT INTO loans VALUES
    ('Home Loan', 5.2500),
    ('Car Loan', 12.3456),
    ('Micro Finance', 0.0125);
```

Penjelasan:

- `5.2500` berarti 5.25%
- `12.3456` berarti 12.3456%
- `0.0125` cocok untuk rate yang sangat kecil

Format ini menjaga konsistensi tampilan sekaligus akurasi perhitungan.

#### 3. Untuk Perhitungan Ilmiah

Dalam konteks ilmiah, pilihan tipe data tergantung pada **kebutuhan presisi**.

```sql
-- Jika butuh presisi sempurna (kasus khusus)
CREATE TABLE scientific_data (
    measurement NUMERIC  -- Unbounded untuk fleksibilitas maksimal
);
```

`NUMERIC` tanpa parameter cocok jika:

- Tidak boleh ada kesalahan pembulatan sama sekali
- Nilai sangat sensitif terhadap presisi

Namun, pada praktiknya:

- Kebanyakan perhitungan ilmiah lebih cocok menggunakan floating-point
- Floating-point jauh lebih cepat dan cukup akurat untuk sains dan statistik

Pemilihan ini sebaiknya disesuaikan dengan kebutuhan domain, bukan kebiasaan semata.

#### 4. Hindari Over-Specification dan Under-Specification

Kesalahan yang sering terjadi adalah menentukan parameter `NUMERIC` secara ekstrem, baik terlalu ketat maupun terlalu longgar.

Contoh **terlalu ketat tanpa alasan jelas**:

```sql
-- ❌ OVERKILL
CREATE TABLE simple_prices (
    price NUMERIC(8, 2)  -- Max 999,999.99
);
```

Jika tidak ada aturan bisnis yang membatasi harga sampai angka ini, lebih baik memberi ruang ekstra:

```sql
-- ✅ Better
price NUMERIC(10, 2)
```

Sebaliknya, contoh **terlalu longgar tanpa validasi**:

```sql
-- ❌ TOO LOOSE
CREATE TABLE product_prices (
    price NUMERIC  -- Tidak ada batasan sama sekali
);
```

Pendekatan ini berisiko karena database tidak lagi membantu memvalidasi data.

Solusi yang lebih seimbang:

```sql
-- ✅ Better
price NUMERIC(10, 2)
```

Dengan cara ini, database ikut menjaga kualitas data tanpa mengorbankan fleksibilitas secara berlebihan.

Secara umum, prinsip terbaik adalah: **beri batasan yang masuk akal sesuai kebutuhan bisnis, tidak lebih dan tidak kurang**.

## 3. Hubungan Antar Konsep

**Progression dari INTEGER ke NUMERIC:**

```
INTEGER (Video sebelumnya)
    ↓
    ├─ Fast, accurate, tapi hanya whole numbers
    └─ Kapan butuh fractional numbers?
        ↓
    NUMERIC (Video ini)
        ↓
        ├─ Slow tapi perfectly precise
        ├─ Biasa untuk financial data
        └─ Parameter (precision, scale) untuk control
            ↓
            ├─ Unbounded: Flexibility maksimal
            ├─ Bounded: Validation built-in
            └─ Negative scale: Advanced rounding
```

**Decision tree memilih tipe:**

```
Apakah data berupa angka?
├─ Ya → Lanjut
└─ Tidak → TEXT/VARCHAR

Apakah butuh fractional numbers?
├─ Tidak → INTEGER (SMALLINT/INT/BIGINT)
└─ Ya → Lanjut

Apakah akurasi sempurna MUTLAK diperlukan?
├─ Ya → NUMERIC
└─ Tidak (boleh aproksimasi) → FLOATING-POINT (next video)
```

## 4. Kesimpulan

`NUMERIC` (alias `DECIMAL`) adalah tipe data PostgreSQL untuk bilangan pecahan dengan **presisi sempurna**. Berada di tengah spektrum antara INTEGER (cepat, whole numbers only) dan FLOATING-POINT (cepat, tapi approximate).

**Key points:**

1. **NUMERIC = DECIMAL:** Keduanya identik, hanya alias

2. **Perfectly accurate:** Tidak ada loss precision seperti FLOAT - **wajib untuk financial data**

3. **Trade-off:** Sangat lambat dibanding INTEGER dan FLOAT

4. **Unbounded by default:** `NUMERIC` tanpa parameter bisa lebih besar dari BIGINT

5. **Precision & Scale untuk control:**

   - `NUMERIC(10, 2)`: 10 total digits, 2 setelah decimal
   - Pattern umum untuk uang: `(10, 2)` atau `(15, 2)`
   - Pattern untuk rates: `(5, 4)` atau `(6, 4)`

6. **Negative scale:** Rounding dari kiri decimal point (PostgreSQL 15+)

7. **Best practice:**
   - Use NUMERIC untuk money intead of FLOAT
   - Specify precision untuk production (validation)
   - Don't over-optimize (gunakan reasonable bounds)

**Kapan menggunakan NUMERIC:**

- ✅ Financial data (harga, salary, tax)
- ✅ Interest rates, percentages yang presisi
- ✅ Accounting, billing systems
- ❌ Counters, quantities (gunakan INTEGER)
- ❌ Scientific approximations (gunakan FLOAT - next video)

NUMERIC memberikan **peace of mind** untuk financial applications - presisi sempurna tanpa rounding errors yang bisa menyebabkan discrepancies dalam accounting. Trade-off performance adalah harga yang layak dibayar untuk akurasi finansial.
