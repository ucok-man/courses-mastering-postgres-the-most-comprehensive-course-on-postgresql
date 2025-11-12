# 📝 Intervals (Durasi Waktu) di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang tipe data **interval** di PostgreSQL, yaitu tipe data yang merepresentasikan **durasi waktu** (bukan titik waktu diskrit seperti timestamp). Interval sangat berguna untuk operasi yang melibatkan rentang waktu, seperti memeriksa apakah dua periode waktu overlap, menghitung durasi, atau memeriksa apakah suatu titik waktu berada dalam rentang tertentu. Video ini fokus pada cara membuat dan memformat interval.

## 2. Konsep Utama

### a. Apa itu Interval?

**Interval** adalah tipe data yang merepresentasikan **durasi waktu**, bukan titik waktu spesifik.

**Perbedaan konsep:**

- **Timestamp/Date/Time**: Titik waktu diskrit (contoh: "12 November 2025 pukul 14:30")
- **Interval**: Durasi atau rentang waktu (contoh: "3 hari 5 jam" atau "2 bulan")

**Use case umum:**

- Booking system: Memeriksa apakah booking ruang konferensi overlap
- Scheduling: Menentukan apakah suatu event berada dalam rentang waktu tertentu
- Perhitungan durasi: Menghitung selisih waktu antara dua event
- Range overlap: Memeriksa apakah beberapa range waktu saling tumpang tindih

**Contoh skenario:**

```sql
-- Memeriksa apakah titik waktu ada dalam interval
-- Menghitung overlap antara dua booking
-- Mencegah double-booking
```

### b. Cara Membuat Interval - Format Postgres Standard

Format paling umum adalah **unit + quantity** yang diulang sesuai kebutuhan, kemudian di-cast ke tipe interval.

**Contoh dasar:**

```sql
-- Interval sederhana
SELECT '1 year'::INTERVAL;
-- Output: 1 year

-- Interval kompleks dengan banyak unit
SELECT '1 year 2 months 3 days 4 hours 5 minutes 6 seconds'::INTERVAL;
-- Output: 1 year 2 mons 3 days 04:05:06
```

**Format yang lebih ringkas:**

```sql
-- Menggunakan format HH:MM:SS untuk waktu
SELECT '1 year 2 months 3 days 04:05:06'::INTERVAL;
-- Output sama: 1 year 2 mons 3 days 04:05:06
```

**Unit yang tersedia:**

- `year` / `years`
- `month` / `months` (disingkat `mons`)
- `day` / `days`
- `hour` / `hours`
- `minute` / `minutes`
- `second` / `seconds`

### c. Interval Style - Mengatur Format Output

Seperti `datestyle`, PostgreSQL memiliki setting `intervalstyle` yang mengontrol **format output** interval (bukan input).

**Melihat interval style saat ini:**

```sql
SHOW intervalstyle;
-- Default: postgres
```

#### Style 1: Postgres (Default)

Format yang paling mudah dibaca:

```sql
SET intervalstyle = 'postgres';

SELECT '1 year 2 months 3 days 04:05:06'::INTERVAL;
-- Output: 1 year 2 mons 3 days 04:05:06
```

#### Style 2: ISO 8601 (Abbreviated)

Format ISO yang lebih ringkas dengan singkatan:

```sql
SET intervalstyle = 'iso_8601';

SELECT '1 year 2 months 3 days 04:05:06'::INTERVAL;
-- Output: P1Y2M3DT4H5M6S
```

**Penjelasan format:**

- `P` - Prefix wajib (Period)
- `Y` - Years (tahun)
- `M` - Months (bulan) - sebelum T
- `D` - Days (hari)
- `T` - Time separator (pemisah waktu)
- `H` - Hours (jam)
- `M` - Minutes (menit) - setelah T
- `S` - Seconds (detik)

**Contoh:**

- `P1Y2M3DT4H5M6S` = 1 tahun, 2 bulan, 3 hari, 4 jam, 5 menit, 6 detik
- `P2Y` = 2 tahun
- `PT5H30M` = 5 jam 30 menit

#### Style 3: ISO 8601 (Alternate)

Format yang mirip timestamp dengan prefix P dan separator T:

```sql
-- Format: P<YYYY-MM-DD>T<HH:MM:SS>
SELECT 'P0001-02-03T04:05:06'::INTERVAL;
-- Output (jika style=postgres): 1 year 2 mons 3 days 04:05:06
```

**Struktur:**

- `P` - Prefix wajib
- `YYYY-MM-DD` - Format tanggal (4 digit tahun, 2 digit bulan, 2 digit hari)
- `T` - Separator
- `HH:MM:SS` - Format waktu

**Catatan penting:**

- `0001` bukan tahun 1 Masehi, tapi durasi 1 tahun
- Format ini terlihat seperti timestamp tapi sebenarnya interval

### d. Cara Membuat Interval - Syntax Alternatif

PostgreSQL menyediakan syntax alternatif menggunakan keyword `INTERVAL`:

**Format 1: Unit tunggal dengan quantity**

```sql
-- Syntax: INTERVAL 'quantity unit'
SELECT INTERVAL '2 year';
-- Output: 2 years
```

**Format 2: Range dengan TO**

```sql
-- Syntax: INTERVAL 'value1 TO value2' unit1 TO unit2
SELECT INTERVAL '1-6' YEAR TO MONTH;
-- Output: 1 year 6 months

-- Artinya: dari 1 (year) sampai 6 (month)
```

**Format 3: Konversi dari unit kecil**

```sql
SELECT INTERVAL '6000 second';
-- Output: 01:40:00 (1 jam 40 menit)

-- PostgreSQL otomatis mengkonversi ke unit yang lebih besar
```

**Contoh lainnya:**

```sql
-- 90 menit menjadi 1 jam 30 menit
SELECT INTERVAL '90 minutes';
-- Output: 01:30:00

-- 36 jam menjadi 1 hari 12 jam
SELECT INTERVAL '36 hours';
-- Output: 1 day 12:00:00
```

### e. Penyimpanan Internal Interval

PostgreSQL menyimpan interval dengan cara yang **smart** untuk menghindari masalah dengan variasi panjang bulan dan daylight saving time.

**Mengapa penting:**

```sql
-- Masalah jika disimpan sebagai hari total:
SELECT '2 months 12 days'::INTERVAL;

-- Tidak bisa dikonversi langsung ke hari karena:
-- - Januari = 31 hari
-- - Februari = 28/29 hari
-- - Bulan lain = 30 atau 31 hari
```

**Cara PostgreSQL menyimpan:**

- **Unit diskrit** untuk komponen yang variabel (year, month)
- **Unit kontinyu** untuk komponen yang tetap (day, hour, minute, second)
- Ini memastikan interval tetap akurat saat ditambahkan ke timestamp

**Keuntungan:**

- Interval aman digunakan melewati daylight saving time
- Tetap akurat pada bulan dengan panjang berbeda
- Tetap valid pada leap year

**Ilustrasi konsep:**

```sql
-- Aman: PostgreSQL tahu bahwa "1 month" berbeda tergantung bulan awal
SELECT TIMESTAMP '2024-01-31' + INTERVAL '1 month';
-- Output: 2024-02-29 (leap year, Februari punya 29 hari)

SELECT TIMESTAMP '2025-01-31' + INTERVAL '1 month';
-- Output: 2025-02-28 (bukan leap year)

-- Berbeda dengan jika disimpan sebagai "30 days":
SELECT TIMESTAMP '2024-01-31' + INTERVAL '30 days';
-- Output: 2024-03-01 (tidak sama dengan "1 month")
```

### f. Kapan Interval Berguna

**Use case yang disebutkan:**

1. **Booking overlap detection**: Mencegah double booking
2. **Range queries**: Query berdasarkan rentang waktu
3. **Point-in-range check**: Memeriksa apakah waktu tertentu ada dalam range
4. **Multiple range overlap**: Memeriksa overlap beberapa range sekaligus

**Catatan:**

- Video ini hanya membahas pembuatan interval
- Querying dengan interval akan dibahas di video terpisah
- Meskipun tidak paling umum, interval adalah tool penting untuk time-based operations

### g. Rekomendasi Interval Style

**Dari video:**

- `postgres` adalah default dan sangat readable
- Pilih style yang paling sesuai dengan use case
- Tidak ada rekomendasi kuat untuk satu style tertentu

**Pertimbangan praktis:**

- **postgres**: Paling mudah dibaca manusia
- **iso_8601**: Lebih ringkas, standar internasional
- **iso_8601 alternate**: Mirip timestamp, bisa membingungkan

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
