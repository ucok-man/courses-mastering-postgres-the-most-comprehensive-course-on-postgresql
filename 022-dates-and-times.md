# 📝 PostgreSQL: Tipe Data Date dan Time (Terpisah)

## 1. Ringkasan Singkat

Materi ini membahas penggunaan tipe data `DATE` dan `TIME` di PostgreSQL yang disimpan secara terpisah, kapan sebaiknya menggunakannya, dan kapan sebaiknya **tidak** menggunakannya. Fokus utamanya adalah memahami perbedaan antara menyimpan "discrete point in time" (gunakan `TIMESTAMP`) versus menyimpan date atau time secara independen.

## 2. Konsep Utama

### a. Kapan Menggunakan DATE dan TIME Terpisah vs TIMESTAMP

**Prinsip Dasar:**

- **Gunakan `TIMESTAMP`** (atau `TIMESTAMPTZ`) jika menyimpan **discrete point in time** — suatu momen spesifik dalam suatu waktu.
- **Gunakan `DATE`** jika hanya perlu menyimpan tanggal tanpa waktu (misalnya: tanggal lahir, hari pendaftaran).
- **Gunakan `TIME`** jika hanya perlu menyimpan waktu tanpa konteks tanggal (misalnya: jam buka toko yang sama setiap hari).

**❌ Jangan pisahkan DATE dan TIME** jika sebenarnya Anda menyimpan satu momen waktu yang utuh — ini akan mempersulit pengelolaan data.

**Contoh:**

```sql
-- ✅ BENAR: Menyimpan kapan user login (discrete point in time)
CREATE TABLE user_sessions (
    user_id INT,
    login_at TIMESTAMPTZ  -- Satu kolom untuk date + time
);

-- ❌ SALAH: Memisahkan date dan time untuk discrete point in time
CREATE TABLE user_sessions_bad (
    user_id INT,
    login_date DATE,
    login_time TIME  -- Jangan lakukan ini!
);

-- ✅ BENAR: Menyimpan jam operasional toko (time saja, berlaku setiap hari)
CREATE TABLE store_hours (
    day_of_week TEXT,
    open_time TIME,
    close_time TIME
);

-- ✅ BENAR: Menyimpan tanggal lahir (date saja, waktu tidak relevan)
CREATE TABLE users (
    user_id INT,
    birth_date DATE
);
```

### b. Tipe Data DATE

Tipe data `DATE` adalah tipe yang paling sederhana — tidak ada precision, tidak ada time zone.

**Syntax:**

```sql
CREATE TABLE events (
    event_date DATE
);
```

**Aturan Ambiguous Date:**

- PostgreSQL menggunakan setting `DateStyle` untuk menginterpretasi input tanggal yang ambigu.
- Setting `datestyle = 'MDY'` berarti Month-Day-Year (format Amerika).
- Output selalu dalam format ISO 8601 (YYYY-MM-DD).

**Contoh:**

```sql
-- Cek setting datestyle
SHOW datestyle;  -- Output: ISO, MDY

-- Input ambigu: 01-03-2024
SELECT '01-03-2024'::DATE;
-- Output: 2024-01-03 (diinterpretasi sebagai January 3, bukan March 1)

-- Input eksplisit ISO 8601 (tidak ambigu)
SELECT '2024-03-01'::DATE;
-- Output: 2024-03-01
```

### c. Tipe Data TIME

Tipe data `TIME` menyimpan waktu dalam sehari tanpa konteks tanggal.

**Syntax:**

```sql
TIME [(precision)] [WITHOUT TIME ZONE]
TIME [(precision)] WITH TIME ZONE  -- ❌ TIDAK DIREKOMENDASIKAN
```

**Parameter:**

- `precision`: 0-6 digit untuk fractional seconds (default: mempertahankan precision literal input sampai 6 digit)
- `WITHOUT TIME ZONE` adalah default
- `WITH TIME ZONE` (atau `TIMETZ`) **sangat tidak direkomendasikan**

**Mengapa TIME WITH TIME ZONE Tidak Masuk Akal:**

Menurut dokumentasi PostgreSQL resmi, `TIME WITH TIME ZONE` diimplementasikan hanya untuk compliance dengan SQL standard, tapi PostgreSQL sendiri **tidak merekomendasikan** penggunaannya.

**Alasan:** Waktu dengan time zone tanpa tanggal adalah **ambigu dan tidak berguna** karena:

- Tidak bisa menentukan apakah sedang daylight saving time atau tidak
- Tidak ada konteks tanggal untuk menentukan offset yang benar

**Contoh:**

```sql
-- ✅ Penggunaan TIME yang benar
CREATE TABLE daily_schedule (
    task_name TEXT,
    start_time TIME,
    end_time TIME
);

INSERT INTO daily_schedule VALUES
    ('Morning Meeting', '09:00:00', '10:00:00');

-- Precision fractional seconds
SELECT '12:01:05'::TIME;           -- Output: 12:01:05
SELECT '12:01:05.1234'::TIME;      -- Output: 12:01:05.1234
SELECT '12:01:05.123456'::TIME;    -- Output: 12:01:05.123456
SELECT '12:01:05.123456'::TIME(2); -- Output: 12:01:05.12 (rounded)

-- ❌ Jangan gunakan TIME WITH TIME ZONE
SELECT '12:00:00+07:00'::TIMETZ;   -- Tidak masuk akal tanpa tanggal
```

### d. String Literals dan Magic Constants

PostgreSQL menyediakan beberapa string literal khusus untuk date dan time:

**Magic Strings:**

```sql
-- ALL BALLS (military jargon) = 00:00:00
SELECT 'allballs'::TIME;  -- Output: 00:00:00

-- EPOCH untuk TIMESTAMP (bukan TIME)
SELECT 'epoch'::TIMESTAMP;  -- Output: 1970-01-01 00:00:00

-- Magic date strings
SELECT 'tomorrow'::DATE;
SELECT 'yesterday'::DATE;
SELECT 'infinity'::DATE;
SELECT '-infinity'::DATE;
```

**⚠️ Peringatan:** Jangan gunakan magic strings seperti `'tomorrow'` atau `'yesterday'` dalam stored procedures, triggers, atau function definitions karena bisa di-compile menjadi nilai literal dan menjadi stale (usang).

**Alternatif yang Aman:**

```sql
-- ❌ Jangan di stored procedure/function
CREATE FUNCTION bad_example() RETURNS DATE AS $$
    SELECT 'tomorrow'::DATE;  -- Bisa di-compile jadi literal
$$ LANGUAGE SQL;

-- ✅ Gunakan ini sebagai gantinya
CREATE FUNCTION good_example() RETURNS DATE AS $$
    SELECT CURRENT_DATE + INTERVAL '1 day';
$$ LANGUAGE SQL;

-- Atau lebih sederhana
SELECT CURRENT_DATE + 1;  -- +1 day
SELECT CURRENT_DATE - 1;  -- -1 day (yesterday)
```

### e. Current Time Constants dan Tipe Return-nya

PostgreSQL memiliki beberapa constants untuk mendapatkan waktu/tanggal saat ini, tapi **hati-hati** karena mereka return tipe data yang berbeda:

| Constant            | Tipe Return                   | Catatan                          |
| ------------------- | ----------------------------- | -------------------------------- |
| `CURRENT_DATE`      | `DATE`                        | ✅ Aman digunakan                |
| `CURRENT_TIMESTAMP` | `TIMESTAMP WITH TIME ZONE`    | ✅ Aman digunakan                |
| `CURRENT_TIME`      | `TIME WITH TIME ZONE`         | ❌ **HINDARI** - returns TIMETZ  |
| `LOCALTIME`         | `TIME WITHOUT TIME ZONE`      | ✅ Alternatif untuk CURRENT_TIME |
| `LOCALTIMESTAMP`    | `TIMESTAMP WITHOUT TIME ZONE` | ✅ Jika butuh timestamp tanpa TZ |

**Contoh:**

```sql
-- Cek tipe return dengan pg_typeof()
SELECT pg_typeof(CURRENT_DATE);      -- date
SELECT pg_typeof(CURRENT_TIMESTAMP); -- timestamp with time zone
SELECT pg_typeof(CURRENT_TIME);      -- time with time zone ❌
SELECT pg_typeof(LOCALTIME);         -- time without time zone ✅
SELECT pg_typeof(LOCALTIMESTAMP);    -- timestamp without time zone

-- Demonstrasi output
SELECT CURRENT_DATE;       -- 2025-11-12
SELECT CURRENT_TIMESTAMP;  -- 2025-11-12 14:30:45.123456+00
SELECT CURRENT_TIME;       -- 14:30:45.123456+00 (perhatikan +00)
SELECT LOCALTIME;          -- 14:30:45.123456 (tanpa timezone offset)
```

**⚠️ Peringatan Penting:** Hindari `CURRENT_TIME` karena return `TIME WITH TIME ZONE` yang tidak direkomendasikan. Gunakan `LOCALTIME` sebagai gantinya.

## 3. Hubungan Antar Konsep

```
┌─────────────────────────────────────────────────────┐
│ Apakah data ini "discrete point in time"?          │
└────────────┬─────────────────────┬──────────────────┘
             │                     │
         YES │                     │ NO
             │                     │
             ▼                     ▼
    ┌─────────────────┐   ┌──────────────────────────┐
    │ Gunakan:        │   │ Pertanyaan lanjutan:     │
    │ TIMESTAMP(TZ)   │   │ Data apa yang disimpan?  │
    └─────────────────┘   └────────┬─────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
                    ▼              ▼              ▼
              ┌──────────┐   ┌─────────┐   ┌──────────────┐
              │ Hanya    │   │ Hanya   │   │ Keduanya     │
              │ tanggal  │   │ waktu   │   │ tapi terpisah│
              └────┬─────┘   └────┬────┘   └──────┬───────┘
                   │              │               │
                   ▼              ▼               ▼
              Gunakan:       Gunakan:        ❌ HINDARI
                DATE           TIME          (use TIMESTAMP)
```

**Key Insight:**

- `DATE` dan `TIME` adalah untuk data yang **conceptually independent** dari konteks waktu/tanggal lengkap.
- Jika date dan time Anda describe satu momen yang sama, jangan pisahkan — gunakan `TIMESTAMP`.

## 4. Catatan Tambahan / Insight

### 💡 Tips Praktis

1. **Time Zone Best Practice:**

   - Selalu set database time zone ke UTC: `SET timezone = 'UTC';`
   - Gunakan `TIMESTAMPTZ` untuk discrete points in time
   - Hindari `TIMETZ` dalam kondisi apapun

2. **Validasi Tipe Data:**

   ```sql
   -- Gunakan pg_typeof() untuk memastikan tipe yang benar
   SELECT pg_typeof(your_column) FROM your_table LIMIT 1;
   ```

3. **Case Study: Store Hours**

   ```sql
   -- Contoh yang valid untuk TIME tanpa DATE
   CREATE TABLE store_hours (
       day_of_week TEXT CHECK (day_of_week IN
           ('Monday', 'Tuesday', 'Wednesday', 'Thursday',
            'Friday', 'Saturday', 'Sunday')),
       open_time TIME NOT NULL,
       close_time TIME NOT NULL
   );

   -- Namun jika ada holiday hours, butuh DATE + TIME = TIMESTAMP!
   CREATE TABLE special_hours (
       special_date DATE,
       open_time TIME,
       close_time TIME
   );
   ```

4. **Fractional Seconds Precision:**

   - Default: PostgreSQL mempertahankan precision dari input literal (max 6 digits)
   - Explicitly specify: `TIME(3)` akan round ke 3 decimal places
   - Range: 0-6 digits (microsecond precision)

5. **ISO 8601 Format:**
   - PostgreSQL selalu output DATE dalam format ISO 8601: `YYYY-MM-DD`
   - Input bisa berbagai format, tapi output konsisten

### ⚠️ Common Pitfalls

1. **Memisahkan DATE dan TIME untuk event timestamps** — ini anti-pattern yang umum
2. **Menggunakan `CURRENT_TIME`** — returns `TIMETZ` yang tidak berguna
3. **Menggunakan magic strings di functions** — bisa menjadi stale
4. **Mengasumsikan `TIME` punya konteks timezone** — `TIME` tidak punya konsep timezone yang meaningful

## 5. Kesimpulan

- **`DATE`** cocok untuk data tanggal yang tidak memerlukan waktu (birth dates, deadlines tanpa jam).
- **`TIME`** cocok untuk waktu yang independen dari tanggal spesifik (daily schedules, business hours).
- **`TIMESTAMP(TZ)`** adalah pilihan yang tepat untuk discrete points in time — jangan pisahkan menjadi DATE + TIME.
- **`TIME WITH TIME ZONE`** adalah fitur yang tidak berguna dan harus dihindari — dokumentasi PostgreSQL sendiri tidak merekomendasikannya.
- Gunakan `LOCALTIME` atau `CURRENT_DATE`, hindari `CURRENT_TIME`.
- Selalu validasi tipe data dengan `pg_typeof()` jika tidak yakin.

**Prinsip Emas:** Jika date dan time describe satu momen yang sama dalam waktu, simpan sebagai satu nilai `TIMESTAMP`, bukan dua kolom terpisah.
