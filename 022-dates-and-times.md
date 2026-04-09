# 📝 PostgreSQL: Tipe Data Date dan Time (Terpisah)

## 1. Ringkasan Singkat

Materi ini membahas penggunaan tipe data `DATE` dan `TIME` di PostgreSQL yang disimpan secara terpisah, kapan sebaiknya menggunakannya, dan kapan sebaiknya **tidak** menggunakannya. Fokus utamanya adalah memahami perbedaan antara menyimpan "discrete point in time" (gunakan `TIMESTAMP`) versus menyimpan date atau time secara independen.

## 2. Konsep Utama

### a. Kapan Menggunakan DATE dan TIME Terpisah vs TIMESTAMP

**Prinsip Dasar:**

Pemilihan tipe data waktu sebaiknya disesuaikan dengan makna data yang ingin disimpan, bukan sekadar kebutuhan teknis.

- **Gunakan `TIMESTAMP` (atau `TIMESTAMPTZ`)** ketika Anda ingin menyimpan **discrete point in time**, yaitu satu momen waktu yang spesifik dan utuh (tanggal + jam + menit + detik). Contohnya adalah waktu login user, waktu transaksi, atau waktu event terjadi.
- **Gunakan `DATE`** jika data hanya membutuhkan informasi tanggal tanpa waktu. Biasanya dipakai untuk data yang tidak bergantung pada jam, seperti tanggal lahir atau tanggal pendaftaran.
- **Gunakan `TIME`** jika hanya perlu menyimpan jam tanpa konteks tanggal. Ini cocok untuk pola waktu yang berulang setiap hari, seperti jam buka dan tutup toko.

**❌ Hindari memisahkan `DATE` dan `TIME`** apabila data yang Anda simpan sebenarnya merepresentasikan satu momen waktu yang utuh. Pemisahan ini akan menyulitkan query, perhitungan, dan pemeliharaan data di kemudian hari.

**Contoh:**

```sql
-- ✅ BENAR: Menyimpan kapan user login (discrete point in time)
CREATE TABLE user_sessions (
    user_id INT,
    login_at TIMESTAMPTZ  -- Satu kolom untuk date + time
);
```

Pada contoh di atas, waktu login disimpan sebagai satu nilai utuh. Ini memudahkan pengurutan, perbandingan waktu, dan konversi zona waktu.

```sql
-- ❌ SALAH: Memisahkan date dan time untuk discrete point in time
CREATE TABLE user_sessions_bad (
    user_id INT,
    login_date DATE,
    login_time TIME  -- Jangan lakukan ini!
);
```

Pendekatan ini terlihat fleksibel, tetapi justru menambah kompleksitas. Untuk sekadar membandingkan waktu login antar user, Anda harus menggabungkan dua kolom terlebih dahulu.

```sql
-- ✅ BENAR: Menyimpan jam operasional toko (time saja, berlaku setiap hari)
CREATE TABLE store_hours (
    day_of_week TEXT,
    open_time TIME,
    close_time TIME
);
```

Di sini, `TIME` digunakan dengan tepat karena jam buka dan tutup tidak terikat pada tanggal tertentu dan berlaku berulang setiap minggu.

```sql
-- ✅ BENAR: Menyimpan tanggal lahir (date saja, waktu tidak relevan)
CREATE TABLE users (
    user_id INT,
    birth_date DATE
);
```

Untuk tanggal lahir, informasi jam tidak memiliki arti bisnis, sehingga `DATE` sudah lebih dari cukup.

### b. Tipe Data DATE

Tipe data `DATE` digunakan untuk menyimpan **tanggal saja**, tanpa informasi waktu, tanpa tingkat ketelitian tambahan (precision), dan tanpa pengaruh zona waktu. Karena sifatnya yang sederhana, `DATE` sangat cocok untuk data yang memang tidak membutuhkan jam, menit, atau detik, seperti tanggal lahir, tanggal pendaftaran, atau tanggal sebuah event secara kalender.

#### Syntax

Berikut contoh paling dasar penggunaan tipe data `DATE` di PostgreSQL:

```sql
CREATE TABLE events (
    event_date DATE
);
```

Pada tabel di atas, kolom `event_date` hanya akan menyimpan nilai tanggal. Tidak ada informasi jam yang disimpan, sehingga PostgreSQL otomatis mengabaikan aspek waktu sepenuhnya.

#### Aturan Ambiguous Date

Saat bekerja dengan `DATE`, penting untuk memahami bagaimana PostgreSQL menafsirkan format tanggal yang **ambigu** (tidak jelas urutan hari dan bulannya).

Beberapa aturan penting yang perlu diingat:

- PostgreSQL menggunakan konfigurasi bernama `DateStyle` untuk menentukan bagaimana input tanggal dibaca.
- Jika `datestyle = 'MDY'`, maka format yang digunakan adalah **Month-Day-Year** (gaya Amerika).
- Meskipun input bisa bermacam-macam, **output PostgreSQL selalu ditampilkan dalam format standar ISO 8601**, yaitu `YYYY-MM-DD`.

Artinya, cara Anda menulis tanggal bisa berpengaruh pada hasil yang disimpan, tetapi hasil akhirnya tetap ditampilkan dalam format yang konsisten.

#### Contoh

```sql
-- Cek setting datestyle
SHOW datestyle;  -- Output: ISO, MDY
```

Perintah di atas menunjukkan bahwa PostgreSQL menggunakan format **MDY** untuk membaca input tanggal yang ambigu.

```sql
-- Input ambigu: 01-03-2024
SELECT '01-03-2024'::DATE;
-- Output: 2024-01-03 (diinterpretasi sebagai January 3, bukan March 1)
```

Karena `datestyle` diset ke `MDY`, PostgreSQL membaca `01-03-2024` sebagai **bulan 01, tanggal 03**, bukan sebaliknya. Inilah yang membuat format seperti ini berbahaya jika tidak dikontrol dengan baik.

```sql
-- Input eksplisit ISO 8601 (tidak ambigu)
SELECT '2024-03-01'::DATE;
-- Output: 2024-03-01
```

Dengan menggunakan format **ISO 8601**, urutan tahun–bulan–tanggal sudah jelas dan tidak ambigu. Ini adalah praktik terbaik yang sangat disarankan, terutama ketika bekerja dengan sistem internasional atau data yang bersumber dari berbagai tempat.

### c. Tipe Data TIME

Tipe data `TIME` digunakan untuk menyimpan **waktu dalam satu hari**, tanpa informasi tanggal apa pun. Artinya, nilai `TIME` hanya merepresentasikan jam, menit, detik, dan opsional fractional seconds, tetapi **tidak terikat pada hari tertentu**. Tipe ini sangat cocok untuk kebutuhan yang bersifat berulang setiap hari, seperti jadwal aktivitas, jam operasional, atau jam kerja.

#### Syntax

Secara umum, PostgreSQL menyediakan beberapa variasi deklarasi tipe `TIME`:

```sql
TIME [(precision)] [WITHOUT TIME ZONE]
TIME [(precision)] WITH TIME ZONE  -- ❌ TIDAK DIREKOMENDASIKAN
```

Pada praktiknya, hampir semua penggunaan `TIME` sebaiknya menggunakan bentuk default, yaitu `TIME WITHOUT TIME ZONE`.

#### Parameter

Beberapa parameter penting yang perlu dipahami saat menggunakan `TIME`:

- `precision` menentukan jumlah digit **fractional seconds**, dengan rentang 0 sampai 6 digit.
- Jika tidak ditentukan, PostgreSQL akan mempertahankan ketelitian sesuai input literal, maksimal hingga 6 digit.
- `WITHOUT TIME ZONE` adalah nilai default dan merupakan pilihan yang benar untuk hampir semua kasus.
- `WITH TIME ZONE` (atau `TIMETZ`) tersedia, tetapi **sangat tidak direkomendasikan**.

#### Mengapa TIME WITH TIME ZONE Tidak Masuk Akal

Menurut dokumentasi resmi PostgreSQL, tipe `TIME WITH TIME ZONE` ada semata-mata untuk memenuhi standar SQL, bukan karena kegunaannya secara praktis. Bahkan PostgreSQL sendiri menyarankan agar tipe ini **tidak digunakan**.

Masalah utamanya adalah: **waktu yang memiliki time zone tanpa tanggal adalah ambigu dan tidak bermakna**.

Beberapa alasannya:

- Tanpa tanggal, PostgreSQL tidak bisa menentukan apakah waktu tersebut berada dalam periode daylight saving time atau tidak.
- Tanpa konteks tanggal, offset time zone tidak dapat dihitung secara akurat.
- Akibatnya, nilai tersebut tidak bisa diinterpretasikan secara konsisten dan berpotensi menimbulkan bug logika.

#### Contoh

```sql
-- ✅ Penggunaan TIME yang benar
CREATE TABLE daily_schedule (
    task_name TEXT,
    start_time TIME,
    end_time TIME
);
```

Pada contoh di atas, kolom `start_time` dan `end_time` merepresentasikan jam aktivitas yang berlaku setiap hari, tanpa bergantung pada tanggal tertentu.

```sql
INSERT INTO daily_schedule VALUES
    ('Morning Meeting', '09:00:00', '10:00:00');
```

Data ini menyatakan bahwa meeting dimulai pukul 09:00 dan berakhir pukul 10:00, terlepas dari hari apa pun.

```sql
-- Precision fractional seconds
SELECT '12:01:05'::TIME;           -- Output: 12:01:05
SELECT '12:01:05.1234'::TIME;      -- Output: 12:01:05.1234
SELECT '12:01:05.123456'::TIME;    -- Output: 12:01:05.123456
SELECT '12:01:05.123456'::TIME(2); -- Output: 12:01:05.12 (rounded)
```

Contoh di atas menunjukkan bagaimana PostgreSQL menangani fractional seconds. Jika precision dibatasi, nilai akan **dibulatkan**, bukan dipotong.

```sql
-- ❌ Jangan gunakan TIME WITH TIME ZONE
SELECT '12:00:00+07:00'::TIMETZ;   -- Tidak masuk akal tanpa tanggal
```

Walaupun sintaks ini valid, hasilnya tidak memiliki makna yang jelas. Tanpa informasi tanggal, offset `+07:00` tidak bisa dievaluasi dengan benar, sehingga penggunaan tipe ini sebaiknya dihindari sepenuhnya.

### d. String Literals dan Magic Constants

PostgreSQL menyediakan beberapa **string literal khusus** yang bisa langsung dikonversi menjadi nilai date atau time. String-string ini sering disebut sebagai _magic constants_ karena memiliki arti bawaan yang sudah dipahami oleh PostgreSQL, tanpa perlu perhitungan manual.

#### Magic Strings

Beberapa magic string yang umum digunakan antara lain:

```sql
-- ALL BALLS (military jargon) = 00:00:00
SELECT 'allballs'::TIME;  -- Output: 00:00:00
```

`'allballs'` adalah istilah lama (military jargon) yang merepresentasikan awal hari, yaitu tepat pukul **00:00:00**.

```sql
-- EPOCH untuk TIMESTAMP (bukan TIME)
SELECT 'epoch'::TIMESTAMP;  -- Output: 1970-01-01 00:00:00
```

`'epoch'` mengacu pada **Unix epoch**, yaitu 1 Januari 1970 pukul 00:00:00. Perlu diperhatikan bahwa magic string ini berlaku untuk `TIMESTAMP`, **bukan untuk `TIME`**.

```sql
-- Magic date strings
SELECT 'tomorrow'::DATE;
SELECT 'yesterday'::DATE;
SELECT 'infinity'::DATE;
SELECT '-infinity'::DATE;
```

Contoh di atas menunjukkan beberapa magic string untuk tipe `DATE`:

- `'tomorrow'` menghasilkan tanggal besok relatif terhadap hari ini
- `'yesterday'` menghasilkan tanggal kemarin
- `'infinity'` dan `'-infinity'` digunakan untuk merepresentasikan tanggal yang sangat jauh di masa depan atau masa lalu

Magic string ini terlihat praktis, tetapi ada risiko tersembunyi jika digunakan secara sembarangan.

#### Peringatan Penggunaan

**⚠️ Jangan gunakan magic strings seperti `'tomorrow'` atau `'yesterday'` di dalam stored procedures, triggers, atau function definitions.**

Alasannya adalah karena PostgreSQL dapat **meng-compile** string tersebut menjadi nilai literal saat fungsi dibuat. Akibatnya, nilai yang seharusnya dinamis bisa menjadi **stale (usang)** dan tidak lagi merepresentasikan tanggal relatif yang diharapkan.

#### Alternatif yang Aman

Perhatikan contoh berikut:

```sql
-- ❌ Jangan di stored procedure/function
CREATE FUNCTION bad_example() RETURNS DATE AS $$
    SELECT 'tomorrow'::DATE;  -- Bisa di-compile jadi literal
$$ LANGUAGE SQL;
```

Fungsi di atas terlihat benar, tetapi berbahaya. Jika fungsi ini dibuat hari ini, maka `'tomorrow'` bisa dikompilasi menjadi tanggal tetap (misalnya 2026-01-23), dan **tidak akan berubah** ketika fungsi dipanggil di hari lain.

Sebagai gantinya, gunakan ekspresi berbasis waktu saat runtime:

```sql
-- ✅ Gunakan ini sebagai gantinya
CREATE FUNCTION good_example() RETURNS DATE AS $$
    SELECT CURRENT_DATE + INTERVAL '1 day';
$$ LANGUAGE SQL;
```

Pendekatan ini memastikan bahwa nilai yang dihasilkan selalu dihitung **saat fungsi dijalankan**, bukan saat fungsi dibuat.

Untuk kebutuhan yang lebih sederhana, PostgreSQL bahkan menyediakan sintaks yang lebih ringkas:

```sql
SELECT CURRENT_DATE + 1;  -- +1 day
SELECT CURRENT_DATE - 1;  -- -1 day (yesterday)
```

Cara ini jauh lebih aman, mudah dibaca, dan bebas dari risiko nilai tanggal yang usang akibat proses kompilasi.

### e. Current Time Constants dan Tipe Return-nya

PostgreSQL menyediakan beberapa **constant bawaan** untuk mengambil tanggal dan waktu saat ini. Sekilas terlihat mirip, tetapi yang sering menjadi jebakan adalah **tipe data yang mereka kembalikan berbeda-beda**. Jika salah pilih, Anda bisa tanpa sadar menggunakan tipe yang justru tidak direkomendasikan.

Memahami perbedaan ini penting agar skema database dan logika query tetap konsisten dan aman.

#### Ringkasan Constant dan Tipe Return

Berikut gambaran singkat mengenai constant yang paling sering digunakan beserta tipe return-nya:

- `CURRENT_DATE` mengembalikan **`DATE`** dan aman digunakan karena hanya berisi tanggal.
- `CURRENT_TIMESTAMP` mengembalikan **`TIMESTAMP WITH TIME ZONE`**, cocok untuk merepresentasikan momen waktu yang spesifik.
- `CURRENT_TIME` mengembalikan **`TIME WITH TIME ZONE`**, dan **sebaiknya dihindari**.
- `LOCALTIME` mengembalikan **`TIME WITHOUT TIME ZONE`**, sehingga lebih masuk akal untuk penggunaan harian.
- `LOCALTIMESTAMP` mengembalikan **`TIMESTAMP WITHOUT TIME ZONE`**, berguna jika Anda memang tidak ingin menyimpan informasi zona waktu.

#### Mengecek Tipe Return Secara Langsung

Untuk memastikan tipe data yang dikembalikan, PostgreSQL menyediakan fungsi `pg_typeof()`:

```sql
-- Cek tipe return dengan pg_typeof()
SELECT pg_typeof(CURRENT_DATE);      -- date
SELECT pg_typeof(CURRENT_TIMESTAMP); -- timestamp with time zone
SELECT pg_typeof(CURRENT_TIME);      -- time with time zone ❌
SELECT pg_typeof(LOCALTIME);         -- time without time zone ✅
SELECT pg_typeof(LOCALTIMESTAMP);    -- timestamp without time zone
```

Dari hasil di atas, terlihat jelas bahwa `CURRENT_TIME` mengembalikan `time with time zone`, yang sebelumnya sudah dibahas sebagai tipe yang problematis karena tidak memiliki konteks tanggal.

#### Contoh Output dan Perbedaannya

```sql
-- Demonstrasi output
SELECT CURRENT_DATE;       -- 2025-11-12
SELECT CURRENT_TIMESTAMP;  -- 2025-11-12 14:30:45.123456+00
SELECT CURRENT_TIME;       -- 14:30:45.123456+00 (perhatikan +00)
SELECT LOCALTIME;          -- 14:30:45.123456 (tanpa timezone offset)
```

Perhatikan perbedaannya:

- `CURRENT_DATE` hanya menampilkan tanggal.
- `CURRENT_TIMESTAMP` menampilkan tanggal, waktu, dan offset zona waktu (`+00`).
- `CURRENT_TIME` menampilkan waktu **dengan offset zona waktu**, yang membuatnya kurang berguna tanpa konteks tanggal.
- `LOCALTIME` menampilkan waktu yang sama, tetapi **tanpa informasi zona waktu**, sehingga lebih konsisten dan aman.

#### Peringatan Penting

**⚠️ Hindari penggunaan `CURRENT_TIME`.**
Constant ini mengembalikan `TIME WITH TIME ZONE`, tipe yang tidak direkomendasikan dan berpotensi membingungkan.

Jika Anda hanya membutuhkan waktu saat ini tanpa tanggal, **gunakan `LOCALTIME`**. Jika membutuhkan momen waktu yang utuh, **gunakan `CURRENT_TIMESTAMP`**.

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
