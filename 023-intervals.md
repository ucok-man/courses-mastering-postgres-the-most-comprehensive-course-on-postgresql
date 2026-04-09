# 📝 Intervals (Durasi Waktu) di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang tipe data **interval** di PostgreSQL, yaitu tipe data yang merepresentasikan **durasi waktu** (bukan titik waktu diskrit seperti timestamp). Interval sangat berguna untuk operasi yang melibatkan rentang waktu, seperti memeriksa apakah dua periode waktu overlap, menghitung durasi, atau memeriksa apakah suatu titik waktu berada dalam rentang tertentu. Video ini fokus pada cara membuat dan memformat interval.

## 2. Konsep Utama

### a. Apa itu Interval?

**Interval** adalah tipe data yang digunakan untuk merepresentasikan **durasi atau rentang waktu**, bukan sebuah titik waktu tertentu. Artinya, interval menjawab pertanyaan _“berapa lama?”_ atau _“dalam rentang waktu apa?”_, bukan _“kapan tepatnya?”_.

Konsep ini sangat penting ketika kita bekerja dengan sistem yang berkaitan dengan jadwal, booking, atau perhitungan waktu, karena banyak masalah waktu tidak bisa diselesaikan hanya dengan timestamp tunggal.

#### Perbedaan Konsep Interval vs Timestamp

Agar lebih jelas, kita bedakan dulu dengan tipe waktu yang lebih umum:

- **Timestamp / Date / Time**
  Merepresentasikan **titik waktu spesifik**.
  Contoh:

  > 12 November 2025 pukul 14:30

- **Interval**
  Merepresentasikan **durasi atau rentang waktu**.
  Contoh:

  > 3 hari 5 jam
  > 2 bulan
  > dari 10:00 sampai 12:00

Dengan kata lain:

- Timestamp → _kapan sesuatu terjadi_
- Interval → _berapa lama_ atau _dalam rentang waktu apa_

#### Kenapa Interval Penting?

Dalam banyak kasus nyata, kita tidak cukup hanya tahu waktu mulai atau waktu selesai saja. Kita perlu memahami **hubungan antar rentang waktu**.

Beberapa use case umum:

- **Booking system**
  Memastikan tidak ada dua booking yang saling tumpang tindih (overlap), misalnya pemesanan ruang meeting.
- **Scheduling**
  Mengecek apakah sebuah event berlangsung di dalam rentang waktu tertentu.
- **Perhitungan durasi**
  Menghitung selisih waktu antara dua kejadian, misalnya durasi kerja atau lama sewa.
- **Range overlap**
  Menentukan apakah dua atau lebih interval waktu saling bertabrakan.

#### Contoh Skenario dalam Sistem Booking

Bayangkan kita punya sistem booking ruang konferensi. Setiap booking memiliki:

- waktu mulai (`start_time`)
- waktu selesai (`end_time`)

Secara logika, ini membentuk **interval waktu**.

Misalnya ada booking lama:

- 10:00 – 12:00

Lalu ada booking baru:

- 11:00 – 13:00

Walaupun waktu mulai dan selesai berbeda, kedua booking ini **overlap**, sehingga booking baru harus ditolak.

#### Contoh Konsep dalam SQL

Berikut contoh komentar SQL yang menggambarkan penggunaan interval dalam kasus nyata:

```sql
-- Memeriksa apakah sebuah titik waktu berada di dalam interval
-- Menghitung apakah dua booking saling overlap
-- Mencegah terjadinya double-booking pada sistem reservasi
```

Dalam implementasi sebenarnya, kita biasanya:

- membandingkan `start_time` dan `end_time`
- memastikan interval baru **tidak bertabrakan** dengan interval yang sudah ada

Sebagai mentor teknis, penting untuk diingat bahwa **interval bukan sekadar data tambahan**, tetapi konsep fundamental dalam desain sistem berbasis waktu. Jika konsep ini dipahami dengan baik sejak awal, banyak bug terkait jadwal, booking, dan perhitungan waktu bisa dihindari sejak tahap desain.

### b. Cara Membuat Interval - Format Postgres Standard

Di PostgreSQL, interval biasanya dibuat menggunakan **string literal** yang berisi pasangan **jumlah (quantity) + satuan waktu (unit)**, lalu di-_cast_ ke tipe `INTERVAL`. Format ini fleksibel dan mudah dibaca, sehingga sering dipakai dalam query sehari-hari.

Secara konsep, PostgreSQL akan membaca string tersebut, memecahnya menjadi unit-unit waktu, lalu menyimpannya sebagai satu nilai interval yang terstruktur.

#### Format Dasar Interval

Format paling umum adalah menuliskan **jumlah diikuti satuan waktu**, dan bisa diulang beberapa kali dalam satu string.

Contoh interval sederhana:

```sql
-- Interval sederhana
SELECT '1 year'::INTERVAL;
-- Output: 1 year
```

Pada contoh ini:

- `'1 year'` adalah string
- `::INTERVAL` memberitahu PostgreSQL untuk mengonversinya ke tipe interval
- Hasilnya adalah interval berdurasi **1 tahun**

#### Interval dengan Banyak Unit

Interval juga bisa terdiri dari **lebih dari satu unit waktu** dalam satu ekspresi.

```sql
-- Interval kompleks dengan banyak unit
SELECT '1 year 2 months 3 days 4 hours 5 minutes 6 seconds'::INTERVAL;
-- Output: 1 year 2 mons 3 days 04:05:06
```

Penjelasan hasilnya:

- PostgreSQL menggabungkan semua unit menjadi satu interval
- `months` ditampilkan sebagai `mons` (singkatan bawaan PostgreSQL)
- Jam, menit, dan detik diformat ulang menjadi `HH:MM:SS`
- `4 hours 5 minutes 6 seconds` menjadi `04:05:06`

Perlu dicatat bahwa **urutan unit tidak terlalu kaku**, selama formatnya valid dan bisa dikenali oleh PostgreSQL.

#### Format yang Lebih Ringkas

Untuk bagian waktu (jam, menit, detik), PostgreSQL mendukung format **HH:MM:SS** yang lebih ringkas dan sering lebih mudah dibaca.

```sql
-- Menggunakan format HH:MM:SS untuk waktu
SELECT '1 year 2 months 3 days 04:05:06'::INTERVAL;
-- Output sama: 1 year 2 mons 3 days 04:05:06
```

Pada contoh ini:

- Bagian tanggal (`1 year 2 months 3 days`) ditulis seperti biasa
- Bagian waktu langsung ditulis dalam format jam
- Hasil akhirnya **identik** dengan contoh sebelumnya

Format ini sangat berguna ketika interval berasal dari input pengguna atau hasil perhitungan jam kerja, durasi event, dan sejenisnya.

#### Unit Waktu yang Tersedia

PostgreSQL mendukung berbagai satuan waktu untuk interval, baik dalam bentuk tunggal maupun jamak:

- `year` / `years`
- `month` / `months` (akan ditampilkan sebagai `mons`)
- `day` / `days`
- `hour` / `hours`
- `minute` / `minutes`
- `second` / `seconds`

Semua unit ini bisa dikombinasikan sesuai kebutuhan. Sebagai mentor teknis, penting untuk diingat bahwa pemilihan unit akan memengaruhi cara PostgreSQL menyimpan dan menampilkan interval, terutama saat berurusan dengan bulan dan tahun yang panjang harinya bisa bervariasi.

Dengan memahami format standar ini, kamu akan lebih percaya diri saat bekerja dengan perhitungan waktu, baik untuk query sederhana maupun logika bisnis yang lebih kompleks.

### c. Interval Style - Mengatur Format Output

Seperti halnya `datestyle`, PostgreSQL juga menyediakan pengaturan bernama `intervalstyle` yang berfungsi untuk mengatur **format tampilan (output)** dari data bertipe `INTERVAL`. Penting dipahami sejak awal bahwa pengaturan ini **tidak memengaruhi cara input ditulis**, melainkan hanya bagaimana interval ditampilkan saat hasil query dikembalikan.

Ini sangat berguna ketika:

- hasil query akan dibaca manusia,
- data interval dikirim ke sistem lain,
- atau harus mengikuti standar tertentu seperti ISO 8601.

#### Melihat Interval Style yang Aktif

Untuk mengetahui style interval yang sedang digunakan oleh PostgreSQL, kita bisa menjalankan:

```sql
SHOW intervalstyle;
-- Default: postgres
```

Secara default, PostgreSQL menggunakan style `postgres`, karena formatnya paling mudah dibaca oleh manusia.

#### Style 1: Postgres (Default)

Style `postgres` adalah format standar bawaan PostgreSQL. Format ini verbose, jelas, dan sangat cocok untuk kebutuhan debugging atau query harian.

```sql
SET intervalstyle = 'postgres';

SELECT '1 year 2 months 3 days 04:05:06'::INTERVAL;
-- Output: 1 year 2 mons 3 days 04:05:06
```

Penjelasan:

- Setiap unit waktu ditampilkan secara eksplisit
- `months` disingkat menjadi `mons`
- Bagian jam, menit, dan detik digabung dalam format `HH:MM:SS`
- Format ini sangat mudah dipahami tanpa perlu pengetahuan khusus

Karena keterbacaannya tinggi, style ini sering dipakai selama proses development.

#### Style 2: ISO 8601 (Abbreviated)

Style `iso_8601` menggunakan standar internasional ISO 8601 dengan format yang jauh lebih ringkas dan terstruktur. Style ini sering digunakan untuk pertukaran data antar sistem atau API.

```sql
SET intervalstyle = 'iso_8601';

SELECT '1 year 2 months 3 days 04:05:06'::INTERVAL;
-- Output: P1Y2M3DT4H5M6S
```

Walaupun terlihat lebih “padat”, format ini sebenarnya sangat konsisten dan bisa diparse dengan mudah oleh mesin.

##### Penjelasan Struktur Format ISO 8601

- `P` → Prefix wajib, singkatan dari _Period_
- `Y` → Years (tahun)
- `M` → Months (bulan), **sebelum `T`**
- `D` → Days (hari)
- `T` → Time separator, penanda bagian waktu
- `H` → Hours (jam)
- `M` → Minutes (menit), **setelah `T`**
- `S` → Seconds (detik)

Perhatikan bahwa huruf `M` memiliki dua arti:

- sebelum `T` = bulan
- setelah `T` = menit

##### Contoh Pembacaan Interval ISO 8601

- `P1Y2M3DT4H5M6S`
  Artinya: 1 tahun, 2 bulan, 3 hari, 4 jam, 5 menit, 6 detik
- `P2Y`
  Artinya: 2 tahun
- `PT5H30M`
  Artinya: 5 jam 30 menit

Format ini sangat cocok ketika interval dikirim sebagai string ke frontend atau layanan eksternal.

#### Style 3: ISO 8601 (Alternate)

PostgreSQL juga mendukung format ISO 8601 alternatif yang tampilannya **mirip timestamp**, namun tetap merepresentasikan interval.

```sql
-- Format: P<YYYY-MM-DD>T<HH:MM:SS>
SELECT 'P0001-02-03T04:05:06'::INTERVAL;
-- Output (jika style=postgres): 1 year 2 mons 3 days 04:05:06
```

##### Struktur Format Alternate

- `P` → Prefix wajib (Period)
- `YYYY-MM-DD` → Bagian tanggal sebagai durasi
  - `YYYY` = tahun (4 digit)
  - `MM` = bulan (2 digit)
  - `DD` = hari (2 digit)

- `T` → Separator antara tanggal dan waktu
- `HH:MM:SS` → Bagian waktu

##### Catatan Penting

- Angka `0001` **bukan** berarti tahun 1 Masehi
  Itu berarti **durasi 1 tahun**
- Walaupun bentuknya menyerupai timestamp, format ini tetap dianggap sebagai **interval**, bukan titik waktu

Sebagai mentor teknis, penting untuk menekankan bahwa pemilihan `intervalstyle` sebaiknya disesuaikan dengan konteks penggunaan. Untuk debugging dan analisis manual, gunakan `postgres`. Untuk integrasi sistem atau standar data, `iso_8601` biasanya pilihan yang lebih aman dan konsisten.

### d. Cara Membuat Interval - Syntax Alternatif

Selain menggunakan string yang di-_cast_ dengan `::INTERVAL`, PostgreSQL juga menyediakan **syntax alternatif** yang lebih eksplisit dengan keyword `INTERVAL`. Pendekatan ini sering dianggap lebih “formal” dan jelas, terutama saat menulis query yang ingin mudah dipahami oleh developer lain.

Secara konsep, semua cara ini tetap menghasilkan tipe data `INTERVAL`. Perbedaannya hanya pada **cara penulisannya**, bukan pada hasil akhirnya.

#### Format 1: Unit Tunggal dengan Quantity

Format paling sederhana menggunakan keyword `INTERVAL` diikuti oleh string berisi **jumlah dan satuan waktu**.

```sql
-- Syntax: INTERVAL 'quantity unit'
SELECT INTERVAL '2 year';
-- Output: 2 years
```

Penjelasan:

- `INTERVAL '2 year'` berarti interval berdurasi 2 tahun
- PostgreSQL secara otomatis menampilkan `year` dalam bentuk jamak (`years`)
- Format ini sangat cocok untuk interval sederhana dan mudah dibaca

#### Format 2: Range dengan `TO`

PostgreSQL juga mendukung format interval berbentuk **range**, menggunakan keyword `TO`. Format ini berguna ketika kita ingin menuliskan durasi dengan struktur yang lebih ketat.

```sql
-- Syntax: INTERVAL 'value1-value2' unit1 TO unit2
SELECT INTERVAL '1-6' YEAR TO MONTH;
-- Output: 1 year 6 months
```

Makna dari query di atas:

- `YEAR TO MONTH` menentukan rentang unit yang digunakan
- `1-6` berarti:
  - `1` untuk unit pertama (YEAR)
  - `6` untuk unit kedua (MONTH)

Dengan kata lain, interval ini dibaca sebagai **1 tahun 6 bulan**. Format ini sering dipakai pada kebutuhan yang membutuhkan konsistensi struktur, misalnya parsing data terformat.

#### Format 3: Konversi Otomatis dari Unit Kecil

Salah satu keunggulan PostgreSQL adalah kemampuannya **mengonversi unit kecil ke unit yang lebih besar secara otomatis**.

```sql
SELECT INTERVAL '6000 second';
-- Output: 01:40:00
```

Penjelasan hasil:

- 6000 detik = 100 menit
- 100 menit = 1 jam 40 menit
- PostgreSQL menampilkannya dalam format `HH:MM:SS`

Ini sangat membantu karena kita tidak perlu menghitung dan memecah durasi secara manual.

#### Contoh Konversi Interval Lainnya

Berikut beberapa contoh tambahan untuk memperjelas perilaku konversi otomatis:

```sql
-- 90 menit menjadi 1 jam 30 menit
SELECT INTERVAL '90 minutes';
-- Output: 01:30:00
```

- 90 menit dikonversi menjadi format jam dan menit
- Output ditampilkan sebagai `01:30:00`

```sql
-- 36 jam menjadi 1 hari 12 jam
SELECT INTERVAL '36 hours';
-- Output: 1 day 12:00:00
```

- 36 jam = 24 jam + 12 jam
- PostgreSQL mengekspresikannya sebagai kombinasi hari dan jam

Sebagai mentor teknis, penting untuk memahami bahwa PostgreSQL selalu berusaha menampilkan interval dalam bentuk yang **paling masuk akal dan mudah dibaca**. Dengan memahami berbagai syntax alternatif ini, kamu bisa memilih gaya penulisan interval yang paling sesuai dengan konteks query, kebutuhan bisnis, dan keterbacaan kode tim.

### e. Penyimpanan Internal Interval

PostgreSQL menyimpan data bertipe `INTERVAL` dengan cara yang **cerdas dan kontekstual**, bukan sekadar sebagai total detik atau total hari. Pendekatan ini sengaja dirancang untuk menghindari berbagai masalah klasik dalam perhitungan waktu, seperti **perbedaan panjang bulan**, **leap year**, dan **daylight saving time**.

Pemahaman bagian ini penting, karena menjelaskan _mengapa_ interval di PostgreSQL sering berperilaku “lebih pintar” dibanding perhitungan waktu manual.

#### Mengapa Penyimpanan Interval Itu Penting?

Bayangkan jika interval disimpan hanya sebagai total hari.

```sql
SELECT '2 months 12 days'::INTERVAL;
```

Secara kasat mata, interval di atas terlihat sederhana. Namun, PostgreSQL **tidak bisa dan tidak boleh** langsung mengonversi `2 months` menjadi jumlah hari tetap, karena:

- Januari memiliki 31 hari
- Februari bisa 28 atau 29 hari
- Bulan lain bisa 30 atau 31 hari
- Leap year mengubah panjang Februari

Jika `2 months` dipaksa menjadi “60 hari”, maka hasil perhitungannya akan sering keliru tergantung konteks tanggal awal.

#### Cara PostgreSQL Menyimpan Interval

Untuk menghindari masalah tersebut, PostgreSQL memisahkan penyimpanan interval ke dalam dua jenis komponen:

- **Unit diskrit (tidak tetap)**
  Digunakan untuk unit yang panjangnya bisa berubah:
  - year
  - month

- **Unit kontinyu (tetap)**
  Digunakan untuk unit yang panjangnya konsisten:
  - day
  - hour
  - minute
  - second

Dengan pendekatan ini:

- `1 month` tetap disimpan sebagai “1 bulan”, bukan dikonversi ke hari
- `30 days` tetap dianggap 30 hari, bukan 1 bulan

Pemisahan ini memastikan bahwa interval selalu dihitung **berdasarkan konteks timestamp awal**.

#### Keuntungan Pendekatan Ini

Pendekatan penyimpanan internal seperti ini memberikan beberapa keuntungan besar:

- Interval tetap akurat saat melewati **daylight saving time**
- Tidak rusak oleh **perbedaan panjang bulan**
- Tetap valid pada **leap year**
- Aman digunakan untuk perhitungan tanggal jangka panjang

Ini adalah alasan mengapa interval di PostgreSQL sangat cocok untuk sistem penjadwalan dan perhitungan waktu yang kompleks.

#### Ilustrasi Konsep dengan Contoh Nyata

Perhatikan contoh berikut:

```sql
-- PostgreSQL tahu bahwa "1 month" bergantung pada bulan awal
SELECT TIMESTAMP '2024-01-31' + INTERVAL '1 month';
-- Output: 2024-02-29
```

Penjelasan:

- Tahun 2024 adalah leap year
- Februari memiliki 29 hari
- PostgreSQL menyesuaikan hasilnya secara kontekstual

Sekarang bandingkan dengan tahun non-leap:

```sql
SELECT TIMESTAMP '2025-01-31' + INTERVAL '1 month';
-- Output: 2025-02-28
```

Hasilnya berbeda, meskipun intervalnya sama, karena konteks tanggal awalnya berbeda.

Bandingkan lagi dengan interval berbasis hari:

```sql
SELECT TIMESTAMP '2024-01-31' + INTERVAL '30 days';
-- Output: 2024-03-01
```

Di sini:

- `30 days` selalu berarti 30 hari penuh
- Tidak peduli bulan apa atau apakah leap year
- Hasilnya **berbeda** dengan `1 month`

Sebagai mentor teknis, poin penting yang perlu diingat adalah:
**“1 month” dan “30 days” bukanlah hal yang sama di PostgreSQL.**
Perbedaan ini bukan bug, melainkan desain yang sengaja dibuat agar perhitungan waktu tetap akurat dan masuk akal di dunia nyata.

### f. Kapan Interval Berguna

Tipe data `INTERVAL` akan terasa sangat berguna ketika kita mulai bekerja dengan **waktu sebagai rentang**, bukan sekadar sebagai titik tunggal. Banyak masalah di dunia nyata sebenarnya tidak bisa diselesaikan hanya dengan membandingkan timestamp, tetapi membutuhkan pemahaman tentang **durasi dan overlap waktu**.

Berikut beberapa use case utama di mana interval menjadi alat yang tepat dan elegan.

#### 1. Booking Overlap Detection

Salah satu penggunaan interval yang paling umum adalah **mencegah double booking**.

Contohnya pada sistem:

- pemesanan ruang meeting
- booking hotel
- jadwal dokter
- reservasi lapangan

Setiap booking biasanya memiliki:

- waktu mulai
- waktu selesai

Dua booking dianggap bermasalah jika interval waktunya **saling tumpang tindih**. Dengan interval, kita bisa memodelkan waktu booking sebagai rentang yang jelas, lalu memeriksa apakah ada overlap sebelum menyimpan data baru.

Secara konsep:

- booking A: 10:00 – 12:00
- booking B: 11:00 – 13:00
  → terjadi overlap, booking harus ditolak

#### 2. Range Queries (Query Berdasarkan Rentang Waktu)

Interval juga sangat membantu saat kita ingin melakukan query data berdasarkan **rentang waktu tertentu**.

Contoh kasus:

- mencari semua event yang berlangsung dalam 7 hari ke depan
- mengambil transaksi dalam 30 hari terakhir
- menampilkan promo yang aktif selama periode tertentu

Alih-alih menghitung tanggal secara manual, interval memungkinkan kita mengekspresikan logika waktu secara lebih deklaratif, misalnya “sekarang + 7 hari” atau “tanggal hari ini – 1 bulan”.

#### 3. Point-in-Range Check

Use case lain yang sangat umum adalah **mengecek apakah suatu titik waktu berada di dalam sebuah rentang**.

Contoh pertanyaan bisnis:

- Apakah promo masih aktif saat ini?
- Apakah user melakukan aksi saat jam operasional?
- Apakah sebuah event sedang berlangsung pada waktu tertentu?

Dengan interval, sebuah rentang waktu bisa didefinisikan dengan jelas, lalu sebuah timestamp dicek apakah berada di dalam rentang tersebut.

Secara konsep:

- range: 09:00 – 17:00
- point: 14:30
  → point berada di dalam range

#### 4. Multiple Range Overlap

Dalam sistem yang lebih kompleks, kita sering perlu memeriksa **lebih dari dua interval sekaligus**.

Contoh kasus:

- mencari semua booking yang bertabrakan dengan jadwal tertentu
- mengecek konflik jadwal antar beberapa event
- validasi jadwal kerja karyawan dengan shift yang saling bersinggungan

Interval memudahkan proses ini karena setiap rentang waktu direpresentasikan secara konsisten, sehingga overlap antar banyak range bisa dianalisis dengan lebih sistematis.

#### Catatan Penting

Beberapa hal penting yang perlu diingat terkait pembahasan interval:

- Fokus materi ini adalah **cara membuat dan memahami interval**
- Cara melakukan query, filtering, dan overlap detection dengan interval biasanya dibahas terpisah karena topiknya cukup luas
- Walaupun interval bukan tipe data yang paling sering dipakai sehari-hari, ia adalah **tool yang sangat penting** untuk semua operasi yang berbasis waktu

Sebagai mentor teknis, pesan utamanya adalah:
jika kamu bekerja dengan jadwal, durasi, atau rentang waktu, memahami interval akan membuat desain sistemmu lebih bersih, lebih aman, dan jauh lebih minim bug terkait waktu.

### g. Rekomendasi Interval Style

Pemilihan `intervalstyle` sebaiknya tidak dilakukan secara sembarangan. Walaupun PostgreSQL menyediakan beberapa opsi, masing-masing style memiliki **tujuan dan konteks penggunaan** yang berbeda. Karena itu, keputusan terbaik biasanya bergantung pada siapa yang akan membaca output dan ke mana data tersebut akan digunakan.

#### Rekomendasi Umum dari Video

Beberapa poin penting yang ditekankan dalam video:

- `postgres` adalah **default PostgreSQL** dan paling mudah dibaca
- Tidak ada satu style yang dianggap paling benar untuk semua kondisi
- Pilih interval style yang **paling sesuai dengan use case**, bukan sekadar preferensi pribadi

Dengan kata lain, PostgreSQL memberi fleksibilitas, dan tanggung jawab pemilihannya ada pada kita sebagai developer.

#### Pertimbangan Praktis dalam Memilih Interval Style

Agar lebih mudah menentukan pilihan, berikut ringkasan kelebihan dan kekurangan tiap style dalam praktik sehari-hari.

##### `postgres`

- Format paling verbose dan jelas
- Sangat mudah dipahami oleh manusia
- Cocok untuk:
  - debugging
  - query manual
  - log database
  - analisis langsung oleh developer atau DBA

Karena tingkat keterbacaannya tinggi, style ini sering menjadi pilihan aman selama proses development.

##### `iso_8601`

- Format ringkas dan terstandarisasi secara internasional
- Mudah diparse oleh sistem lain
- Cocok untuk:
  - API
  - pertukaran data antar sistem
  - integrasi dengan frontend atau layanan eksternal

Style ini biasanya lebih disukai ketika output interval akan dikonsumsi oleh mesin, bukan manusia.

##### `iso_8601 alternate`

- Bentuknya menyerupai timestamp
- Menggunakan prefix `P` dan separator `T`
- Berpotensi membingungkan karena terlihat seperti tanggal dan waktu absolut

Style ini sebaiknya digunakan dengan hati-hati, terutama jika tim atau sistem yang membaca data tidak benar-benar memahami bahwa ini adalah **interval, bukan timestamp**.

#### Kesimpulan Praktis

Sebagai mentor teknis, pendekatan yang paling aman adalah:

- gunakan `postgres` untuk keterbacaan dan kejelasan
- gunakan `iso_8601` jika ada kebutuhan standar atau integrasi sistem
- hindari `iso_8601 alternate` kecuali benar-benar dibutuhkan dan dipahami oleh semua pihak

Intinya, `intervalstyle` adalah soal **konteks dan tujuan**, bukan soal benar atau salah.

## 3. Hubungan Antar Konsep

**Flow pemahaman:**

1. **Interval vs Timestamp** → Interval adalah durasi, bukan titik waktu diskrit
2. **Multiple format input** → Fleksibilitas dalam membuat interval (standard, ISO, atau syntax INTERVAL)
3. **Interval style** → Mengontrol output format (tidak mempengaruhi penyimpanan)
4. **Internal storage** → PostgreSQL menyimpan secara smart untuk menangani variasi kalender
5. **Future querying** → Interval akan sangat powerful saat digunakan dalam query (dibahas terpisah)

**Hierarki konsep:**

```
Interval (Tipe Data Durasi)
├── Input Format
│   ├── Postgres Standard: '1 year 2 months 3 days'
│   ├── ISO 8601: 'P1Y2M3D'
│   └── Syntax INTERVAL: INTERVAL '1-6' YEAR TO MONTH
├── Output Style
│   ├── postgres (default, readable)
│   ├── iso_8601 (abbreviated)
│   └── iso_8601 alternate (timestamp-like)
├── Internal Storage
│   ├── Discrete units (year, month)
│   └── Continuous units (day, hour, minute, second)
└── Use Cases
    ├── Booking overlap
    ├── Range queries
    └── Duration calculations
```

**Relasi dengan tipe waktu lain:**

- **TIMESTAMP**: Titik waktu + Interval = Titik waktu baru
- **DATE**: Date + Interval = Date baru
- **TIME**: Time + Interval = Time baru

## 4. Kesimpulan

Interval adalah tipe data PostgreSQL yang merepresentasikan **durasi waktu** dan sangat berguna untuk operasi yang melibatkan rentang waktu. Fitur utamanya:

**Key takeaways:**

- **Bukan titik waktu**: Interval adalah durasi, seperti "3 hari" bukan "3 Januari"
- **Fleksibel dalam input**: Banyak cara untuk membuat interval
- **Customizable output**: Interval style dapat disesuaikan (postgres, iso_8601, dll)
- **Smart storage**: PostgreSQL menyimpan dengan cara yang menghindari masalah kalender
- **Powerful untuk queries**: Meskipun video ini hanya cover basics, interval sangat powerful untuk time-based queries

**Kapan menggunakan interval:**

- Booking systems dan scheduling
- Range overlap detection
- Duration calculations
- Time-based business logic yang kompleks

**Next step:**
Video ini hanya membahas cara membuat interval. Kekuatan sebenarnya dari interval akan terlihat saat digunakan dalam querying operations (akan dibahas di video terpisah dalam course).
