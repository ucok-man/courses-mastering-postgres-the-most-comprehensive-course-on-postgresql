# 📝 Time Zones di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang penanganan **time zones** (zona waktu) di PostgreSQL, yang merupakan salah satu topik yang paling membingungkan dalam pemrograman database. Pembahasan mencakup best practices dalam menyimpan dan menampilkan waktu, perbedaan antara named time zones dan offset, serta jebakan umum yang harus dihindari saat bekerja dengan zona waktu.

## 2. Konsep Utama

### a. Kompleksitas Time Zones

Time zones itu rumit karena sifatnya **politis, bukan matematis**. Beberapa hal yang membuat time zones rumit:

- **Daylight Saving Time (DST)**: Tidak semua region menerapkan DST, dan yang menerapkan pun memiliki tanggal peralihan yang berbeda-beda
- **Perubahan Offset**: Offset dari UTC bisa berubah tergantung apakah sedang DST atau tidak
- **Ambiguitas**: Saat konversi, kita harus tahu apakah tanggal yang disimpan dalam DST atau tidak

### b. Best Practice #1: Simpan Semuanya dalam UTC

**Rekomendasi utama**: Simpan semua timestamp dalam UTC selama mungkin, dan konversi ke zona waktu lokal user hanya saat akan ditampilkan (di tahap paling akhir).

**Cara setting time zone di PostgreSQL:**

```sql
-- Melihat time zone saat ini
SHOW timezone;

-- Setting time zone untuk session saat ini
SET timezone = 'America/Chicago';

-- Setting time zone di level database
ALTER DATABASE demo SET timezone TO 'UTC';
```

**Hierarki time zone setting** (dari yang paling umum ke paling spesifik):

1. **Cluster level**: Di file konfigurasi PostgreSQL (`postgresql.conf`)
2. **Database level**: Setting menggunakan `ALTER DATABASE`
3. **Session level**: Setting menggunakan `SET timezone`

> **Catatan**: Session setting akan direset saat koneksi terputus, kembali ke setting database atau cluster.

### c. Perilaku `TIMESTAMP WITH TIME ZONE` (TIMESTAMPTZ)

PostgreSQL secara otomatis melakukan konversi saat bekerja dengan tipe data `TIMESTAMPTZ`:

1. **Saat data masuk**: PostgreSQL mengkonversi ke UTC untuk disimpan
2. **Saat data keluar**: PostgreSQL mengkonversi dari UTC ke time zone session untuk ditampilkan

**Contoh:**

```sql
-- Session di UTC
SET timezone = 'UTC';
SELECT '2025-01-31 11:30:00'::TIMESTAMPTZ;
-- Hasil: 2025-01-31 11:30:00+00

-- Session di America/Chicago
SET timezone = 'America/Chicago';
SELECT '2025-01-31 11:30:00'::TIMESTAMPTZ;
-- Hasil: 2025-01-31 05:30:00-06
-- PostgreSQL otomatis mengkonversi untuk display
```

Jika literal tidak memiliki time zone identifier, PostgreSQL akan menggunakan time zone dari session sebagai default.

### d. Best Practice #2: Gunakan Named Time Zones

**SANGAT PENTING**: Selalu gunakan **named time zones** (seperti `'America/Chicago'`), bukan offset (seperti `'-06:00'`).

**Alasan:**

1. Named time zones otomatis menangani Daylight Saving Time
2. Menghindari kesalahan tanda (positif/negatif) yang bisa menyebabkan error arah konversi hingga 12 jam

**Cara konversi ke time zone lokal:**

```sql
SET timezone = 'UTC';

SELECT
  '2025-01-31 11:30:00'::TIMESTAMPTZ AS utc_time,
  '2025-01-31 11:30:00'::TIMESTAMPTZ
    AT TIME ZONE 'America/Chicago' AS chicago_time;
```

**Output:**

```
utc_time: 2025-01-31 11:30:00+00
chicago_time: 2025-01-31 05:30:00  (tanpa timezone identifier)
```

> **Penting**: Hasil `AT TIME ZONE` adalah `TIMESTAMP WITHOUT TIME ZONE`. Ini seharusnya hanya digunakan di tahap akhir pipeline untuk display ke user.

### e. Jebakan: Penggunaan Hour Offset

PostgreSQL mendukung 3 cara specify time zone:

1. ✅ **Full time zone name**: `'America/Chicago'` (RECOMMENDED)
2. ⚠️ **Time zone abbreviation**: `'CST'`, `'CDT'` (harus handle DST manual)
3. ❌ **POSIX-style offset**: `'-06:00'` (BERBAHAYA!)

**Masalah dengan POSIX-style offset:**

Menurut dokumentasi PostgreSQL, POSIX-style menggunakan **tanda kebalikan** dari ISO 8601:

- **ISO 8601** (standar internasional): Wilayah barat Greenwich menggunakan tanda **negatif** (-)
- **POSIX**: Wilayah barat Greenwich menggunakan tanda **positif** (+)

**Contoh kesalahan:**

```sql
SET timezone = 'UTC';

SELECT
  '2025-01-31 11:30:00'::TIMESTAMPTZ AS base,
  '2025-01-31 11:30:00'::TIMESTAMPTZ
    AT TIME ZONE 'America/Chicago' AS named,        -- 05:30 ✅
  '2025-01-31 11:30:00'::TIMESTAMPTZ
    AT TIME ZONE '-06:00' AS offset_wrong,          -- 17:30 ❌ (salah 12 jam!)
  '2025-01-31 11:30:00'::TIMESTAMPTZ
    AT TIME ZONE INTERVAL '-6 hours' AS interval_ok; -- 05:30 ✅
```

**Penjelasan:**

- `'America/Chicago'` → Benar: 05:30 (11:30 - 6 jam)
- `'-06:00'` → Salah: 17:30 (interpreted as POSIX, jadi 11:30 + 6 jam)
- `INTERVAL '-6 hours'` → Benar: 05:30 (menggunakan ISO 8601)

### f. Melihat Daftar Time Zones

PostgreSQL menyediakan view untuk melihat semua time zone yang tersedia:

```sql
-- Melihat semua time zones
SELECT * FROM pg_timezone_names;

-- Filter berdasarkan nama
SELECT * FROM pg_timezone_names
WHERE name ILIKE '%chicago%';
```

**Output example:**

```
name              | abbrev | utc_offset | is_dst
------------------+--------+------------+--------
America/Chicago   | CDT    | -05:00     | t
```

Kolom `abbrev` dan `utc_offset` akan berubah otomatis tergantung apakah saat ini dalam periode DST atau tidak.

## 3. Hubungan Antar Konsep

Alur kerja yang benar dalam menangani time zones:

1. **Storage**: Simpan semua timestamp sebagai `TIMESTAMPTZ` dalam UTC
2. **Konfigurasi**: Set database/session ke UTC untuk menghindari konversi otomatis yang tidak diinginkan
3. **Processing**: Lakukan semua operasi date/time dalam UTC
4. **Display**: Hanya di tahap akhir, gunakan `AT TIME ZONE` dengan **named time zone** untuk konversi ke zona waktu lokal user
5. **Never**: Jangan gunakan bare hour offset (`'-06:00'`) karena bisa salah arah

**Diagram Alur:**

```
User Input → Convert to UTC → Store in DB (UTC)
                                    ↓
                            Process in UTC
                                    ↓
                    Retrieve as UTC from DB
                                    ↓
              AT TIME ZONE 'America/Chicago'
                                    ↓
                          Display to User
```

## 4. Kesimpulan

Time zones di PostgreSQL memang kompleks, tapi bisa diatasi dengan mengikuti dua prinsip utama:

1. **Simpan dalam UTC, konversi di akhir**: Keep everything in UTC as long as possible, convert to user's local time zone only at display time
2. **Gunakan named time zones**: Always use named time zones like `'America/Chicago'`, never use bare hour offsets like `'-06:00'`

Dengan mengikuti prinsip ini, Anda akan terhindar dari bug yang umum terjadi seperti:

- Salah arah konversi (12 jam off)
- Tidak handle DST dengan benar
- Konversi ganda yang tidak diinginkan

**Remember**: PostgreSQL's time zone handling is powerful, but you need to be explicit and careful. When in doubt, stay in UTC!
