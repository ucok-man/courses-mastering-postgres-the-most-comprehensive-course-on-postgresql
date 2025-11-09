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

Fractional precision melakukan **ROUNDING**, bukan truncating:

```sql
SELECT
    now() AS original,                    -- 13:01:01.923456
    now()::TIMESTAMPTZ(0) AS rounded,     -- 13:01:02 (dibulatkan ke atas!)
    date_trunc('second', now()) AS truncated;  -- 13:01:01 (dipotong ke bawah)
```

**Implikasi:**

- `TIMESTAMPTZ(0)` bisa menghasilkan timestamp **di masa depan** (jika rounded up)
- Jika butuh truncate (potong), gunakan `date_trunc('second', value)`

**Kapan ini masalah:**

```sql
-- Bisa bermasalah untuk audit/logging
INSERT INTO audit_log (action_time)
VALUES (now()::TIMESTAMPTZ(0));
-- Jika now() = 10:30:59.8, akan tersimpan: 10:31:00 (masa depan!)

-- Solusi: gunakan date_trunc
INSERT INTO audit_log (action_time)
VALUES (date_trunc('second', now()));
-- Akan tersimpan: 10:30:59 (masa lalu, lebih aman)
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

## 4. Catatan Tambahan / Insight

### 💡 Best Practices

**1. Selalu gunakan TIMESTAMPTZ:**

```sql
-- ✅ RECOMMENDED
CREATE TABLE users (
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ
);

-- ❌ AVOID (unless you have very specific reason)
CREATE TABLE users (
    created_at TIMESTAMP  -- loses timezone context!
);
```

**2. Gunakan ISO 8601 dalam application code:**

```javascript
// ✅ GOOD - ISO 8601
const timestamp = "2024-01-15T13:45:30Z";

// ❌ BAD - ambiguous
const timestamp = "1/15/2024 1:45 PM";
```

**3. Set default timezone di connection level:**

```sql
-- Di psql atau connection string
SET timezone = 'UTC';  -- Recommended untuk consistency

-- Atau set per-user/per-database
ALTER DATABASE mydb SET timezone = 'UTC';
ALTER USER myuser SET timezone = 'America/New_York';
```

**4. Fractional seconds untuk use case tertentu:**

```sql
-- High-frequency logging/events
CREATE TABLE api_logs (
    request_time TIMESTAMPTZ(6)  -- microsecond precision
);

-- User-facing timestamps
CREATE TABLE posts (
    published_at TIMESTAMPTZ(0)  -- second precision cukup
);
```

### ⚠️ Common Pitfalls

**1. Mixing TIMESTAMP dan TIMESTAMPTZ:**

```sql
-- ⚠️ Bisa menyebabkan confusion
CREATE TABLE mixed_up (
    created TIMESTAMP,      -- no timezone context
    updated TIMESTAMPTZ     -- with timezone context
);

-- Comparison bisa unexpected
WHERE created > updated  -- comparing apples to oranges!
```

**2. Rounding unexpected behavior:**

```sql
-- Bisa menghasilkan timestamp di masa depan!
INSERT INTO logs (happened_at)
VALUES (now()::TIMESTAMPTZ(0));
-- Jika now() = 10:59:59.8, tersimpan: 11:00:00
```

**3. Bergantung pada datestyle:**

```sql
-- ❌ DANGEROUS - behavior berubah based on setting
SELECT '1/3/2024'::DATE;

-- ✅ SAFE - always explicit
SELECT '2024-01-03'::DATE;
```

**4. Lupa time zone saat compare:**

```sql
-- Query: "Show events from today"
-- ❌ WRONG - assumes server timezone
WHERE event_time::DATE = CURRENT_DATE;

-- ✅ CORRECT - explicit timezone
WHERE event_time AT TIME ZONE 'America/New_York'
    BETWEEN '2024-01-15 00:00:00' AND '2024-01-15 23:59:59';
```

### 🎯 Advanced Tips

**1. Current timestamp functions:**

```sql
-- Berbagai cara mendapatkan current timestamp
SELECT
    now(),                    -- TIMESTAMPTZ, evaluated at transaction start
    clock_timestamp(),        -- TIMESTAMPTZ, evaluated at statement execution
    CURRENT_TIMESTAMP,        -- sama dengan now()
    transaction_timestamp(),  -- sama dengan now()
    statement_timestamp();    -- similar to clock_timestamp()

-- Perbedaan penting:
BEGIN;
    SELECT now();              -- 2024-01-15 10:00:00
    -- sleep 5 seconds
    SELECT now();              -- 2024-01-15 10:00:00 (SAME!)
    SELECT clock_timestamp();  -- 2024-01-15 10:00:05 (DIFFERENT!)
COMMIT;
```

**2. Timezone conversion on the fly:**

```sql
-- Convert ke timezone tertentu
SELECT
    created_at,                                    -- Original
    created_at AT TIME ZONE 'UTC',                 -- Force UTC
    created_at AT TIME ZONE 'America/New_York',    -- Convert to NYC
    created_at AT TIME ZONE 'Asia/Jakarta';        -- Convert to Jakarta
```

**3. Extract parts:**

```sql
SELECT
    EXTRACT(YEAR FROM now()),
    EXTRACT(MONTH FROM now()),
    EXTRACT(DAY FROM now()),
    EXTRACT(HOUR FROM now()),
    EXTRACT(TIMEZONE FROM now());  -- timezone offset in seconds
```

### 🌍 Time Zone Trivia

**Time zones dengan offset unusual:**

- India: UTC+05:30 (setengah jam!)
- Nepal: UTC+05:45 (45 menit!)
- Chatham Islands (NZ): UTC+12:45 (45 menit!)
- Australia Central: UTC+09:30 (setengah jam!)

**Mengapa ini penting:**

- Tidak semua time zone offset sejam penuh
- Format ISO 8601 support: `+05:30`, `+12:45`
- PostgreSQL handle semua ini dengan benar via TIMESTAMPTZ

### 📊 Storage Considerations

```sql
-- Semua ini ukuran storage-nya SAMA
CREATE TABLE storage_test (
    ts_no_tz    TIMESTAMP,        -- 8 bytes
    ts_with_tz  TIMESTAMPTZ,      -- 8 bytes (timezone tidak stored per-row!)
    ts_prec_0   TIMESTAMPTZ(0),   -- 8 bytes
    ts_prec_6   TIMESTAMPTZ(6)    -- 8 bytes
);
```

**Insight:**

- TIMESTAMPTZ **tidak menyimpan** timezone per-row
- Timezone hanya digunakan untuk konversi I/O
- Internal storage: semua dalam UTC
- Precision tidak mempengaruhi storage size (tetap 8 bytes)

## 5. Kesimpulan

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
