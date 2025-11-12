# 📝 Tipe Data Timestamp di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang **timestamp** di PostgreSQL—cara menyimpan tanggal dan waktu dalam satu kolom. Ada dua jenis utama: `TIMESTAMP` (without time zone) dan `TIMESTAMP WITH TIME ZONE` (timestamptz). Video menjelaskan perbedaan keduanya, mengapa sebaiknya selalu menggunakan timestamp with time zone, format ISO 8601 untuk menghindari ambiguitas, handling fractional seconds, dan konversi Unix timestamp. Topik time zone akan dibahas lebih mendalam di video terpisah karena kompleksitasnya.

## 2. Konsep Utama

### a. Dua Jenis Timestamp di PostgreSQL

PostgreSQL memiliki dua tipe untuk menyimpan date + time:

**1. TIMESTAMP (tanpa time zone) - default:**

```sql
CREATE TABLE events (
    created_at TIMESTAMP  -- sama dengan TIMESTAMP WITHOUT TIME ZONE
);
```

**Karakteristik:**

- ❌ **Tidak melakukan konversi apapun**
- ❌ Membuang semua informasi time zone
- ❌ Tidak pernah translate nilai
- ⚠️ Menyimpan nilai "as is" tanpa context

**2. TIMESTAMP WITH TIME ZONE (direkomendasikan):**

```sql
CREATE TABLE events (
    created_at TIMESTAMP WITH TIME ZONE,  -- atau gunakan alias: TIMESTAMPTZ
    updated_at TIMESTAMPTZ                -- alias yang lebih singkat
);
```

**Karakteristik:**

- ✅ **Konversi otomatis ke UTC** saat menyimpan
- ✅ **Konversi otomatis ke time zone client** saat membaca
- ✅ Date math lebih mudah dan akurat
- ✅ Handling daylight saving time otomatis

**Perbandingan storage:**

- Keduanya memiliki **range yang sama**
- Keduanya memiliki **underlying storage yang sama**
- Perbedaan hanya pada **behavior konversi**

**Contoh perbedaan behavior:**

```sql
-- WITHOUT time zone: menyimpan literal apa adanya
INSERT INTO events_no_tz (created_at)
VALUES ('2024-01-15 10:00:00');
-- Disimpan: 2024-01-15 10:00:00 (tidak tahu time zone apa)

-- WITH time zone: konversi ke UTC
SET timezone = 'America/New_York';  -- UTC-5
INSERT INTO events_with_tz (created_at)
VALUES ('2024-01-15 10:00:00');
-- Disimpan internal: 2024-01-15 15:00:00 UTC
-- Ditampilkan ke client NYC: 2024-01-15 10:00:00-05
```

### b. Fractional Seconds (Precision)

Timestamp mendukung **0 hingga 6 digit** fractional seconds:

```sql
-- Sintaks: TIMESTAMP(precision)
CREATE TABLE precise_events (
    no_precision TIMESTAMPTZ,           -- default: ambil dari literal (max 6)
    three_digits TIMESTAMPTZ(3),        -- milliseconds: 0.123
    zero_digits  TIMESTAMPTZ(0),        -- no fractional seconds
    six_digits   TIMESTAMPTZ(6)         -- microseconds: 0.123456
);
```

**Contoh penggunaan:**

```sql
SELECT
    now()::TIMESTAMPTZ,         -- 2024-01-15 13:01:01.923456-05
    now()::TIMESTAMPTZ(3),      -- 2024-01-15 13:01:01.923-05
    now()::TIMESTAMPTZ(0);      -- 2024-01-15 13:01:02-05 (rounded!)
```

**⚠️ WARNING: Rounding vs Truncating**

Fractional precision melakukan **ROUNDING** (pembulatan standar), bukan truncating:

```sql
-- Rounding mengikuti aturan standar: >= 0.5 dibulatkan ke atas, < 0.5 ke bawah
SELECT
    '13:01:01.499999'::TIMESTAMPTZ(0) AS rounded_down,  -- 13:01:01 (< 0.5, rounded down)
    '13:01:01.500000'::TIMESTAMPTZ(0) AS rounded_up,    -- 13:01:02 (>= 0.5, rounded up)
    '13:01:01.923456'::TIMESTAMPTZ(0) AS rounded_up2;   -- 13:01:02 (>= 0.5, rounded up)

-- Bandingkan dengan date_trunc yang selalu truncate (potong ke bawah)
SELECT
    now() AS original,                         -- 13:01:01.923456
    now()::TIMESTAMPTZ(0) AS rounded,          -- 13:01:02 (rounded using standard rounding)
    date_trunc('second', now()) AS truncated;  -- 13:01:01 (always truncated down)
```

**Implikasi:**

- `TIMESTAMPTZ(0)` menggunakan pembulatan standar: nilai ≥ 0.5 detik akan dibulatkan ke atas
- Jika butuh truncate (potong ke bawah), gunakan `date_trunc('second', value)`
- **PENTING:** Rounding bisa menghasilkan timestamp **di masa depan** jika fractional seconds ≥ 0.5

**Kapan ini masalah:**

```sql
-- Bisa bermasalah untuk audit/logging
INSERT INTO audit_log (action_time)
VALUES (now()::TIMESTAMPTZ(0));
-- Jika now() = 10:30:59.6, akan tersimpan: 10:31:00 (masa depan!)
-- Jika now() = 10:30:59.4, akan tersimpan: 10:30:59 (masa lalu)

-- Solusi: gunakan date_trunc untuk konsistensi (selalu truncate)
INSERT INTO audit_log (action_time)
VALUES (date_trunc('second', now()));
-- Akan tersimpan: 10:30:59 (selalu masa lalu atau sekarang, lebih aman untuk audit)
```

### c. Format Input: ISO 8601 (RECOMMENDED!)

**Format ISO 8601 - The Gold Standard:**

```
YYYY-MM-DD HH:MI:SS[.ffffff][timezone]
```

**Variasi yang valid:**

```sql
-- Dengan T separator (standar formal)
'2024-01-31T13:45:30'

-- Dengan space separator (lebih readable)
'2024-01-31 13:45:30'

-- Dengan fractional seconds
'2024-01-31 13:45:30.123456'

-- Dengan time zone: Zulu (UTC)
'2024-01-31 13:45:30Z'

-- Dengan UTC offset
'2024-01-31 13:45:30-06:00'  -- Central Time
'2024-01-31 13:45:30+05:30'  -- India (setengah jam!)
'2024-01-31 13:45:30+00:00'  -- UTC
```

**Mengapa ISO 8601 penting:**

- ✅ **Tidak ambiguous** - tidak ada interpretasi ganda
- ✅ **Internationally recognized**
- ✅ **Sortable** - bisa sort as string dan hasilnya benar
- ✅ **Machine-readable**

### d. Ambiguous Date Formats (AVOID!)

**Format yang AMBIGUOUS:**

```sql
-- Format 1/3/2024 - apa artinya?
'1/3/2024'
-- USA (MDY): January 3, 2024
-- Rest of world (DMY): March 1, 2024

-- Format 3/1/2024 - apa artinya?
'3/1/2024'
-- USA (MDY): March 1, 2024
-- Rest of world (DMY): January 3, 2024
```

**PostgreSQL handling ambiguity dengan `datestyle`:**

```sql
-- Cek setting saat ini
SHOW datestyle;
-- Result: ISO, DMY
--         ^^^  ^^^
--         |    |__ Ambiguous date interpretation (DMY/MDY/YMD)
--         |_______ Output format (ISO/SQL/Postgres/German)

-- Setting ambiguous date format
SET datestyle = 'ISO, MDY';  -- USA style
SELECT '1/3/2024'::DATE;      -- Interpreted as: January 3, 2024

SET datestyle = 'ISO, DMY';  -- International style
SELECT '1/3/2024'::DATE;      -- Interpreted as: March 1, 2024
```

**Output format options:**

```sql
-- ISO format (RECOMMENDED)
SET datestyle = 'ISO, DMY';
SELECT '2024-01-31'::DATE;
-- Output: 2024-01-31

-- SQL format (misnomer, bukan SQL standard!)
SET datestyle = 'SQL, DMY';
SELECT '2024-01-31'::DATE;
-- Output: 01/31/2024

-- German format
SET datestyle = 'German, DMY';
SELECT '2024-01-31'::DATE;
-- Output: 31.01.2024

-- European (covers German format)
SET datestyle = 'European';
SELECT '2024-01-31'::DATE;
-- Output: 31.01.2024
```

**⚠️ Best Practice:**

```sql
-- ❌ JANGAN LAKUKAN INI - bergantung pada datestyle setting
INSERT INTO events (event_date) VALUES ('1/3/2024');

-- ✅ LAKUKAN INI - explicit dan clear
INSERT INTO events (event_date) VALUES ('2024-01-03');
```

### e. Unix Timestamp Conversion

**Konversi Unix timestamp ke PostgreSQL timestamp:**

```sql
-- Unix timestamp: detik sejak epoch (1970-01-01 00:00:00 UTC)
SELECT to_timestamp(1705329690);
-- Result: 2024-01-15 12:34:50+00

-- Dengan fractional seconds (float)
SELECT to_timestamp(1705329690.123456);
-- Result: 2024-01-15 12:34:50.123456+00

-- Cek tipe data
SELECT pg_typeof(to_timestamp(1705329690));
-- Result: timestamp with time zone
```

**Karakteristik `to_timestamp()`:**

- ✅ Selalu return `TIMESTAMPTZ` (with time zone)
- ✅ Selalu dalam UTC (+00:00)
- ✅ Preserve fractional seconds jika ada
- ✅ Correct by design (Unix timestamp is UTC-based)

**Konversi balik ke Unix timestamp:**

```sql
-- Timestamp ke Unix timestamp (epoch)
SELECT EXTRACT(EPOCH FROM now());
-- Result: 1705329690.123456 (float dengan fractional seconds)

SELECT EXTRACT(EPOCH FROM now())::BIGINT;
-- Result: 1705329690 (integer, no fractional)
```

### f. Date vs Time vs Timestamp

PostgreSQL memiliki tipe terpisah:

```sql
CREATE TABLE time_types (
    -- Hanya tanggal (no time)
    birth_date DATE,                -- 2024-01-15

    -- Hanya waktu (no date) - jarang berguna!
    opening_time TIME,              -- 09:00:00

    -- Tanggal + waktu (recommended)
    created_at TIMESTAMPTZ,         -- 2024-01-15 09:00:00+00

    -- Interval/duration
    duration INTERVAL               -- '2 hours 30 minutes'
);
```

**Kapan menggunakan masing-masing:**

- **DATE**: Untuk tanggal saja (birthdate, deadline, event date)
- **TIME**: Jarang berguna (opening hours mungkin, tapi timestamp lebih baik)
- **TIMESTAMPTZ**: Untuk semua timestamp (created_at, updated_at, event_time)
- **INTERVAL**: Untuk duration/selisih waktu

## 3. Hubungan Antar Konsep

**Flow decision tree untuk timestamp:**

```
Butuh menyimpan date + time?
    ↓
Butuh time zone awareness?
    ↓ YES (99% cases)
    TIMESTAMPTZ ← USE THIS!
    ↓
Input format?
    ↓
ISO 8601 (YYYY-MM-DD HH:MI:SS) ← ALWAYS USE THIS!
    ↓
Fractional seconds needed?
    ↓
    - High precision logging: (3) atau (6)
    - User timestamps: (0) atau default
    ↓
Rounding behavior OK?
    ↓ NO
    Use date_trunc('second', value)
```

**Conversion flow:**

```
Unix Timestamp (integer/float)
    ↓ to_timestamp()
TIMESTAMPTZ (UTC)
    ↓ Automatic conversion based on session timezone
Display to user (local timezone)
```

**Ambiguity problem:**

```
Ambiguous format: '1/3/2024'
    ↓
Depends on datestyle setting
    ↓ MDY
January 3, 2024
    ↓ DMY
March 1, 2024

SOLUTION: Use ISO 8601 → '2024-01-03' (unambiguous!)
```

## 4. Kesimpulan

**Key Takeaways:**

1. **Gunakan TIMESTAMPTZ (bukan TIMESTAMP):**

   - ✅ Konversi otomatis ke/dari UTC
   - ✅ Date math lebih mudah
   - ✅ Daylight saving time handled automatically
   - ❌ TIMESTAMP plain kehilangan context time zone

2. **Gunakan ISO 8601 format:**

   - ✅ Format: `YYYY-MM-DD HH:MI:SS[.ffffff][TZ]`
   - ✅ Tidak ambiguous
   - ✅ Internationally recognized
   - ❌ Jangan bergantung pada datestyle setting

3. **Fractional seconds dengan hati-hati:**

   - `TIMESTAMPTZ(0)` untuk user-facing timestamps
   - `TIMESTAMPTZ(3)` atau `(6)` untuk high-precision logging
   - ⚠️ Aware of rounding behavior - gunakan `date_trunc()` jika perlu truncate

4. **Unix timestamp conversion:**

   - `to_timestamp(unix_ts)` untuk convert ke TIMESTAMPTZ
   - `EXTRACT(EPOCH FROM ts)` untuk convert balik ke Unix timestamp
   - Selalu dalam UTC (correct by design)

5. **Avoid ambiguous formats:**
   - Jangan gunakan `1/3/2024` atau `3/1/2024`
   - Selalu explicit dengan `2024-01-03`
   - Set `datestyle = 'ISO, DMY'` atau `'ISO, MDY'` untuk consistency

**Philosophy:**

> "Be explicit. Be clear. Let the database handle complexity (like timezone conversion) while you provide unambiguous inputs."

**Next topic teased:** Time zones akan dibahas lebih mendalam di video terpisah karena incredibly important dan incredibly complex! 🕐🌍
