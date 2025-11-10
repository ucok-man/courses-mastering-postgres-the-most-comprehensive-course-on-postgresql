# 📝 Tipe Data Floating-Point di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tipe data floating-point di PostgreSQL, yaitu `REAL` dan `DOUBLE PRECISION`, yang merupakan tipe data untuk bilangan pecahan dengan **presisi variabel dan tidak exact (approximate)**. Berbeda dengan NUMERIC yang lambat tapi perfect, floating-point sangat cepat tetapi menghasilkan aproksimasi. Video mendemonstrasikan perbedaan performa dan presisi antara floating-point dan NUMERIC melalui benchmarking langsung.

## 2. Konsep Utama

### a. Spektrum Lengkap Tipe Data Numerik

Setelah mempelajari INTEGER dan NUMERIC, sekarang kita melengkapi spektrum dengan floating-point:

**Visualisasi spektrum lengkap:**

```
| Property     | INTEGER          | NUMERIC / DECIMAL     | FLOATING-POINT      |
|--------------|------------------|-----------------------|---------------------|
| Fractional   | ❌ Tidak         | ✅ Ya                 | ✅ Ya               |
| Accuracy     | ✅ Sempurna      | ✅ Sempurna           | ❌ Approximate      |
| Precision    | ✅ Perfect       | ✅ Perfect            | ❌ Dapat hilang     |
| Speed        | ⚡ Sangat cepat   | 🐌 Sangat lambat      | ⚡ Sangat cepat      |

```

**Perbandingan karakteristik:**

| Aspek                   | INTEGER          | NUMERIC        | FLOATING-POINT         |
| ----------------------- | ---------------- | -------------- | ---------------------- |
| **Support fractions**   | ❌               | ✅             | ✅                     |
| **Perfectly accurate**  | ✅               | ✅             | ❌ Approximation       |
| **Speed**               | ⚡⚡⚡ Very fast | 🐌 Very slow   | ⚡⚡⚡ Very fast       |
| **Use case**            | Counters, Money  | Money, finance | Science, IoT, graphics |
| **Math precision loss** | Never            | Never          | **Possible**           |

**Key insight:**

> Floating-point **BUKAN berarti buruk atau salah** - ini adalah **approximation yang seringkali cukup baik**. Tidak cocok untuk financial data, tapi sempurna untuk temperature, pressure, sensor readings, scientific calculations.

### b. Dua Tipe Floating-Point di PostgreSQL

#### **1. REAL (4 bytes)**

**Spesifikasi:**

- **Ukuran:** 4 bytes
- **Range:** 1E-37 sampai 1E+37
  - Artinya: 10^-37 sampai 10^37 (range sangat besar!)
- **Precision:** Minimal 6 digits setelah decimal point
- **Alias:** `FLOAT4`

**Contoh penggunaan:**

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

**Kapan menggunakan REAL:**

- Sensor data (temperature, pressure, humidity)
- Graphics coordinates
- Scientific measurements (tidak critical)
- Ketika range dan speed penting, precision sempurna tidak wajib

#### **2. DOUBLE PRECISION (8 bytes)**

**Spesifikasi:**

- **Ukuran:** 8 bytes (2x lebih besar dari REAL)
- **Range:** 1E-307 sampai 1E+308
  - Range yang "magnificent" - sangat sangat besar!
- **Precision:** Minimal 15 digits setelah decimal point
- **Alias:** `FLOAT8`, `DOUBLE`

**Contoh penggunaan:**

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

**Kapan menggunakan DOUBLE PRECISION:**

- Default choice untuk floating-point (kalau ragu, pilih ini)
- Scientific calculations dengan presisi lebih tinggi
- Machine learning features
- Geographic coordinates dengan presisi tinggi
- Ketika butuh range dan precision lebih besar dari REAL

**Perbandingan REAL vs DOUBLE PRECISION:**

```
╔═══════════════════╦════════════╦════════════════════╦═══════════════╗
║ Tipe              ║ Bytes      ║ Range              ║ Precision     ║
╠═══════════════════╬════════════╬════════════════════╬═══════════════╣
║ REAL (FLOAT4)     ║ 4 bytes    ║ 1E-37 to 1E+37     ║ ≥6 digits     ║
║ DOUBLE PRECISION  ║ 8 bytes    ║ 1E-307 to 1E+308   ║ ≥15 digits    ║
║ (FLOAT8)          ║            ║                    ║               ║
╚═══════════════════╩════════════╩════════════════════╩═══════════════╝
```

### c. Aliases untuk Floating-Point

PostgreSQL menyediakan berbagai alias untuk tipe floating-point:

**Untuk REAL (4 bytes):**

```sql
-- Semua deklarasi ini IDENTIK
CREATE TABLE test1 (value REAL);
CREATE TABLE test2 (value FLOAT4);
CREATE TABLE test3 (value FLOAT(1));   -- FLOAT(1-24) → REAL
CREATE TABLE test4 (value FLOAT(24));  -- Max untuk REAL

-- Verifikasi dengan \d
\d test1  -- Type: real
\d test2  -- Type: real
\d test3  -- Type: real
```

**Untuk DOUBLE PRECISION (8 bytes):**

```sql
-- Semua deklarasi ini IDENTIK
CREATE TABLE test1 (value DOUBLE PRECISION);
CREATE TABLE test2 (value FLOAT8);
CREATE TABLE test3 (value FLOAT(25));  -- FLOAT(25-53) → DOUBLE PRECISION
CREATE TABLE test4 (value FLOAT(49));
CREATE TABLE test5 (value FLOAT(53));  -- Max untuk DOUBLE PRECISION

-- Verifikasi
\d test1  -- Type: double precision
\d test2  -- Type: double precision
```

**Sintaks FLOAT(n):**

```
FLOAT(1-24)  → Converts to REAL (4 bytes)
FLOAT(25-53) → Converts to DOUBLE PRECISION (8 bytes)

Note: Nilai n sebenarnya DIABAIKAN di PostgreSQL
(mirip dengan MySQL yang juga ignore parameter ini)
```

**Best practice:**

```sql
-- ✅ RECOMMENDED: Gunakan nama yang jelas
CREATE TABLE sensors (
    temp REAL,              -- Jelas dan standard
    pressure DOUBLE PRECISION  -- Jelas dan standard
);

-- ✅ ACCEPTABLE: Gunakan FLOAT4/FLOAT8 (eksplisit ukuran)
CREATE TABLE metrics (
    small_value FLOAT4,     -- 4 bytes, jelas
    large_value FLOAT8      -- 8 bytes, jelas
);

-- ⚠️ CONFUSING: Hindari FLOAT(n) (parameter diabaikan)
CREATE TABLE confusing (
    value FLOAT(30)  -- Orang mungkin pikir ini special, padahal sama aja
);
```

### d. Demonstrasi: Imprecision dalam Floating-Point

**Problem fundamental floating-point:**

Floating-point menggunakan **binary representation** untuk menyimpan angka, yang tidak bisa merepresentasikan semua decimal numbers secara exact.

**Contoh imprecision:**

```sql
-- Operasi matematis sederhana
SELECT 7.0::FLOAT8 * 2.0 / 10.0;
-- Expected: 1.4
-- Actual:   1.4000000000000001

-- Ini BUKAN bug PostgreSQL!
-- Ini adalah karakteristik floating-point numbers
```

**Mengapa ini terjadi?**

```
Decimal: 1.4
Binary representation: 1.0110011001100... (repeating)

Computer harus "cut off" di suatu titik
→ Slight inaccuracy terjadi
→ Result: 1.4000000000000001 instead of 1.4
```

**Problem dalam comparison:**

```sql
-- ❌ Strict equality TIDAK AKAN WORK
SELECT (7.0::FLOAT8 * 2.0 / 10.0) = 1.4;
-- Result: false (karena 1.4000000000000001 ≠ 1.4)

-- ✅ SOLUTION: Gunakan epsilon comparison
SELECT ABS((7.0::FLOAT8 * 2.0 / 10.0) - 1.4) < 0.001;
-- Result: true
-- Logic: "Apakah perbedaannya < 0.001? Kalau ya, anggap sama"
```

**Best practice untuk comparison:**

```sql
-- Jangan: Strict equality
WHERE float_column = 1.4  -- ❌ Risky!

-- Lakukan: Epsilon comparison
WHERE ABS(float_column - 1.4) < 0.0001  -- ✅ Safe

-- Atau gunakan range
WHERE float_column BETWEEN 1.3999 AND 1.4001  -- ✅ Safe
```

**Kontras dengan NUMERIC:**

```sql
-- NUMERIC: Perfectly precise
SELECT 7.0::NUMERIC * 2.0 / 10.0;
-- Result: 1.4 (exactly!)

SELECT (7.0::NUMERIC * 2.0 / 10.0) = 1.4;
-- Result: true (perfect equality)

-- Ini kenapa NUMERIC wajib untuk financial data!
```

### e. Benchmarking: Speed Comparison

Video mendemonstrasikan **benchmark nyata** antara FLOAT8 dan NUMERIC menggunakan 20 juta rows.

**Setup benchmark:**

```sql
-- Generate 20 juta rows dengan generate_series()
-- Lakukan operasi matematis pada setiap row

-- Test 1: FLOAT8
SELECT num::FLOAT8 / 3.0::FLOAT8 AS float_result
FROM generate_series(1, 20000000) AS num;
-- Execution time: 2.3 seconds

-- Test 2: NUMERIC
SELECT num::NUMERIC / 3.0::NUMERIC AS numeric_result
FROM generate_series(1, 20000000) AS num;
-- Execution time: 4.5-4.6 seconds
```

**Hasil benchmark:**

```
FLOAT8:  2.3 seconds  ⚡⚡⚡
NUMERIC: 4.5 seconds  🐌

Speed difference: ~2x (FLOAT8 adalah 2x lebih cepat)
atau: NUMERIC adalah 2x lebih lambat
```

**Interpretasi:**

- **FLOAT8 memang signifikan lebih cepat** dari NUMERIC
- Perbedaan menjadi **sangat terasa** pada operasi matematis intensif
- 20 juta operasi: perbedaan ~2 detik (bisa sangat critical untuk aplikasi real-time)

**Catatan tentang benchmark:**

> "Not a very scientific benchmark, but hopefully it's big enough that we'll see a pretty big difference."

Meskipun sederhana, benchmark ini **cukup untuk demonstrate** trade-off utama:

- NUMERIC: Slow tapi accurate
- FLOAT: Fast tapi approximate

### f. Fungsi generate_series(): Bonus Tool

**Pengenalan singkat:**

```sql
-- Generate series angka dari 1 sampai 20
SELECT * FROM generate_series(1, 20);
-- Result:
-- 1
-- 2
-- 3
-- ...
-- 20
```

**Kegunaan:**

- Generate test data on-the-fly (tanpa perlu INSERT)
- Benchmarking tanpa perlu buat table permanent
- Generate sequences untuk testing
- Populate temporary data

**Sintaks lengkap:**

```sql
-- Basic: start to end
generate_series(start, end)

-- With step
generate_series(start, end, step)

-- Contoh
SELECT * FROM generate_series(1, 10, 2);
-- Result: 1, 3, 5, 7, 9

-- Date series
SELECT * FROM generate_series(
    '2024-01-01'::DATE,
    '2024-01-10'::DATE,
    '1 day'::INTERVAL
);
-- Generate 10 hari berturut-turut
```

**Use case dalam benchmark:**

```sql
-- Generate 20 juta rows untuk testing tanpa INSERT
SELECT
    num::FLOAT8 / 3.0::FLOAT8 AS float_result
FROM generate_series(1, 20000000) AS num;

-- Sangat powerful untuk performance testing!
```

### g. Decision Matrix: Kapan Gunakan Apa?

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

## 4. Catatan Tambahan / Insight

### 💡 Tips Praktis

**1. Default choice untuk non-financial fractional data:**

```sql
-- Kalau ragu, gunakan DOUBLE PRECISION
CREATE TABLE measurements (
    value DOUBLE PRECISION  -- Safe default
);

-- Hanya turun ke REAL kalau:
-- - Yakin precision 6 digits cukup
-- - Performance critical
-- - Storage menjadi concern
```

**2. JANGAN PERNAH gunakan floating-point untuk money:**

```sql
-- ❌ DISASTER WAITING TO HAPPEN
CREATE TABLE transactions (
    amount REAL  -- Uang bisa "hilang" karena rounding!
);

-- ✅ ALWAYS use NUMERIC for money
CREATE TABLE transactions (
    amount NUMERIC(10, 2)
);
```

**3. Epsilon comparison pattern:**

```sql
-- Create helper function untuk comparison
CREATE FUNCTION float_equals(a REAL, b REAL, epsilon REAL DEFAULT 0.0001)
RETURNS BOOLEAN AS $$
BEGIN
    RETURN ABS(a - b) < epsilon;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT float_equals(7.0::FLOAT8 * 2.0 / 10.0, 1.4);
-- Result: true
```

**4. Naming convention untuk clarity:**

```sql
-- Eksplisit tentang tipe
CREATE TABLE sensor_readings (
    temp_celsius REAL,              -- Jelas: temperature
    pressure_hpa DOUBLE PRECISION,  -- Jelas: pressure
    reading_timestamp TIMESTAMP
);
```

### ⚠️ Kesalahan Umum

**1. Menggunakan strict equality pada floating-point:**

```sql
-- ❌ HAMPIR SELALU SALAH
WHERE temperature = 23.5

-- ✅ GUNAKAN RANGE
WHERE temperature BETWEEN 23.49 AND 23.51
-- atau
WHERE ABS(temperature - 23.5) < 0.01
```

**2. Mixing types tanpa cast explicit:**

```sql
-- ⚠️ BISA CONFUSING
SELECT 7.0 * 2.0 / 10.0;  -- Tipe apa ini?

-- ✅ EXPLICIT CASTING
SELECT 7.0::FLOAT8 * 2.0 / 10.0;  -- Jelas: FLOAT8
SELECT 7.0::NUMERIC * 2.0 / 10.0;  -- Jelas: NUMERIC
```

**3. Over-precision untuk use case yang tidak perlu:**

```sql
-- ❌ OVERKILL: Weather app dengan 15 digits precision
CREATE TABLE weather (
    temperature DOUBLE PRECISION  -- 23.456789012345678°C ??
);

-- ✅ REASONABLE: 6 digits cukup untuk weather
CREATE TABLE weather (
    temperature REAL  -- 23.4567°C sudah sangat cukup
);
```

**4. Tidak aware tentang imprecision:**

```sql
-- ❌ CODE YANG AKAN BERMASALAH
CREATE TABLE products (
    price REAL  -- Approximation untuk harga??
);

SELECT SUM(price) FROM products WHERE category = 'electronics';
-- Total bisa "sedikit meleset" karena accumulation of errors!

-- ✅ CORRECT
CREATE TABLE products (
    price NUMERIC(10, 2)
);
```

### 🎯 Analogies & Mental Models

**Analogi tukang ukur:**

```
INTEGER = Penggaris (hanya cm bulat, sangat cepat)
    "1 cm, 2 cm, 3 cm..."

NUMERIC = Mikrometer (presisi sempurna, lambat)
    "1.00000 cm, 1.00001 cm..."

FLOAT = Pita ukur (cepat, kira-kira)
    "sekitar 1 cm, sekitar 2.5 cm..."
    (good enough untuk most construction!)
```

**Analogi uang:**

```
NUMERIC = Bank accounting system
    Every cent must be exact!
    $10.00 ≠ $10.0000001

FLOAT = Estimasi budget
    "Sekitar $10-ish"
    $9.99999 ≈ $10 (close enough... NOT!)

DON'T USE FLOAT FOR MONEY!
```

**Analogi sensor:**

```
Temperature sensor: 23.456789°C

REAL: "23.4567°C"
    Good enough! Kita ga perlu precision lebih.

DOUBLE PRECISION: "23.456789012345°C"
    Overkill untuk simple temperature sensor.

NUMERIC: "23.456789000000000°C"
    Perfect tapi lambat - unnecessary untuk sensor!
```

### 📊 Tabel Perbandingan Lengkap

**Complete numeric types comparison:**

```
╔════════════════╦═══════╦══════════════╦═══════════╦══════════════╗
║ Type           ║ Bytes ║ Precision    ║ Speed     ║ Use Case     ║
╠════════════════╬═══════╬══════════════╬═══════════╬══════════════╣
║ SMALLINT       ║ 2     ║ Perfect      ║ ⚡⚡⚡       ║ Small counts ║
║ INTEGER        ║ 4     ║ Perfect      ║ ⚡⚡⚡       ║ General IDs  ║
║ BIGINT         ║ 8     ║ Perfect      ║ ⚡⚡⚡       ║ Large IDs    ║
║ NUMERIC        ║ Var   ║ Perfect      ║ 🐌        ║ Money!       ║
║ REAL (FLOAT4)  ║ 4     ║ ~6 digits    ║ ⚡⚡⚡       ║ IoT, sensors ║
║ DOUBLE (FLOAT8)║ 8     ║ ~15 digits   ║ ⚡⚡⚡       ║ Science, ML  ║
╚════════════════╩═══════╩══════════════╩═══════════╩══════════════╝
```

**Performance summary (20M operations):**

```
INTEGER:  ~1.5s   (fastest, tapi whole only)
FLOAT8:   ~2.3s   (fast dengan fractional)
NUMERIC:  ~4.5s   (2x slower, tapi perfect)
```

### 🔍 Advanced: Kenapa Floating-Point "Inexact"?

**Technical explanation (simplified):**

```
Decimal: 0.1 (easy untuk manusia)

Binary representation:
0.0001100110011001100110011... (repeating forever!)

Computer storage (terbatas):
0.00011001100110011001100110 (cut off)
                            ↑
                    Precision loss terjadi di sini

Saat di-convert kembali ke decimal:
0.10000000000000001 (bukan 0.1 exact!)
```

**Contoh konkret:**

```sql
-- 0.1 + 0.2 = ???
SELECT 0.1::FLOAT8 + 0.2::FLOAT8;
-- Result: 0.30000000000000004 (BUKAN 0.3!)

-- Ini BUKAN bug, ini fundamental limitation dari binary floating-point

-- Solution untuk critical comparison:
SELECT ABS((0.1::FLOAT8 + 0.2::FLOAT8) - 0.3) < 0.00001;
-- Result: true (consider "equal" dalam epsilon)
```

**Baca lebih lanjut:**

- IEEE 754 floating-point standard
- "What Every Computer Scientist Should Know About Floating-Point Arithmetic"

### 🔧 Testing Your Understanding

**Quiz mental:**

1. **Kolom untuk temperature sensor reading?**

   - Jawaban: `REAL` (speed matters, approximation OK)

2. **Kolom untuk transaction amount?**

   - Jawaban: `NUMERIC(10, 2)` (money must be exact!)

3. **Kolom untuk GPS latitude/longitude?**

   - Jawaban: `DOUBLE PRECISION` (butuh high precision)

4. **Kenapa `7.0::FLOAT8 * 2.0 / 10.0` bukan exactly 1.4?**

   - Jawaban: Binary representation limitation dari floating-point

5. **Kapan boleh pakai FLOAT untuk financial data?**

   - Jawaban: **NEVER!** Always use NUMERIC

6. **FLOAT8 vs NUMERIC untuk 20M operations: siapa lebih cepat?**
   - Jawaban: FLOAT8 ~2x lebih cepat (2.3s vs 4.5s)

## 5. Kesimpulan

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
