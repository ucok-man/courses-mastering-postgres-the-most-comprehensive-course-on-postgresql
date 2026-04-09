# 📝 Tipe Data Timestamp di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang **timestamp** di PostgreSQL—cara menyimpan tanggal dan waktu dalam satu kolom. Ada dua jenis utama: `TIMESTAMP` (without time zone) dan `TIMESTAMP WITH TIME ZONE` (timestamptz). Video menjelaskan perbedaan keduanya, mengapa sebaiknya selalu menggunakan timestamp with time zone, format ISO 8601 untuk menghindari ambiguitas, handling fractional seconds, dan konversi Unix timestamp. Topik time zone akan dibahas lebih mendalam di video terpisah karena kompleksitasnya.

## 2. Konsep Utama

### a. Dua Jenis Timestamp di PostgreSQL

PostgreSQL menyediakan **dua tipe data** untuk menyimpan kombinasi **tanggal dan waktu (date + time)**. Keduanya terlihat mirip di permukaan, tetapi memiliki **perilaku yang sangat berbeda**, terutama terkait **time zone**. Memahami perbedaan ini penting agar data waktu yang disimpan tetap konsisten dan tidak menimbulkan bug tersembunyi.

#### 1. TIMESTAMP (tanpa time zone) — default

```sql
CREATE TABLE events (
    created_at TIMESTAMP  -- sama dengan TIMESTAMP WITHOUT TIME ZONE
);
```

Tipe ini sering digunakan tanpa disadari karena merupakan **default** ketika kita menulis `TIMESTAMP`. Namun, perlu dipahami bahwa tipe ini **tidak memiliki konsep time zone sama sekali**.

**Karakteristik utama:**

- ❌ Tidak melakukan konversi apa pun
- ❌ Semua informasi time zone dibuang
- ❌ Tidak pernah menerjemahkan nilai berdasarkan lokasi
- ⚠️ Menyimpan nilai waktu **apa adanya** tanpa konteks

Artinya, PostgreSQL hanya menyimpan angka tanggal dan jam persis seperti yang Anda tulis. Database **tidak tahu** apakah waktu tersebut berasal dari Jakarta, New York, atau Tokyo.

Contohnya:

```sql
INSERT INTO events (created_at)
VALUES ('2024-01-15 10:00:00');
```

Nilai yang tersimpan hanyalah:

```
2024-01-15 10:00:00
```

Tanpa informasi tambahan, waktu ini **ambigu**. Jika aplikasi Anda diakses oleh user dari berbagai time zone, tipe ini berpotensi menimbulkan kebingungan dan kesalahan perhitungan.

#### 2. TIMESTAMP WITH TIME ZONE (direkomendasikan)

```sql
CREATE TABLE events (
    created_at TIMESTAMP WITH TIME ZONE,  -- atau alias: TIMESTAMPTZ
    updated_at TIMESTAMPTZ                -- alias yang lebih singkat
);
```

Meskipun namanya “with time zone”, PostgreSQL **tidak benar-benar menyimpan nama time zone**. Yang disimpan adalah **waktu dalam UTC**, lalu PostgreSQL secara cerdas melakukan konversi saat data ditampilkan ke client.

**Karakteristik utama:**

- ✅ Otomatis dikonversi ke **UTC saat disimpan**
- ✅ Otomatis dikonversi ke **time zone client saat dibaca**
- ✅ Perhitungan tanggal dan waktu (date math) lebih akurat
- ✅ Penanganan daylight saving time dilakukan otomatis

Dengan tipe ini, PostgreSQL selalu memiliki **referensi waktu yang konsisten**, yaitu UTC, sehingga aman digunakan pada sistem terdistribusi atau aplikasi global.

#### Perbandingan Storage

Hal menarik yang sering disalahpahami adalah soal penyimpanan data:

- Kedua tipe memiliki **range nilai yang sama**
- Keduanya menggunakan **underlying storage yang sama**
- Perbedaannya **bukan pada ukuran atau performa**, melainkan pada **perilaku konversi**

Jadi, memilih `TIMESTAMPTZ` tidak membuat database Anda lebih berat—hanya membuatnya **lebih benar secara semantik**.

#### Contoh Perbedaan Perilaku

Perhatikan perbedaan berikut untuk memahami efek nyata di lapangan.

**Tanpa time zone (TIMESTAMP WITHOUT TIME ZONE):**

```sql
INSERT INTO events_no_tz (created_at)
VALUES ('2024-01-15 10:00:00');
```

Hasilnya:

- Disimpan: `2024-01-15 10:00:00`
- PostgreSQL tidak tahu itu jam berapa di dunia nyata
- Tidak ada konversi, tidak ada konteks

**Dengan time zone (TIMESTAMP WITH TIME ZONE):**

```sql
SET timezone = 'America/New_York';  -- UTC-5

INSERT INTO events_with_tz (created_at)
VALUES ('2024-01-15 10:00:00');
```

Yang terjadi di balik layar:

- Waktu input dianggap sebagai **10:00 di New York**
- PostgreSQL mengonversinya ke UTC
- Disimpan secara internal sebagai: `2024-01-15 15:00:00 UTC`

Saat data dibaca kembali oleh client dengan time zone New York:

```
2024-01-15 10:00:00-05
```

Namun jika dibaca oleh client di UTC atau Asia/Jakarta, PostgreSQL akan otomatis menampilkan waktu yang sesuai dengan time zone masing-masing.

Inilah alasan mengapa **TIMESTAMPTZ hampir selalu menjadi pilihan yang lebih aman dan direkomendasikan**, terutama untuk aplikasi modern yang berjalan lintas lokasi dan zona waktu.

### b. Fractional Seconds (Precision)

PostgreSQL mendukung **fractional seconds** pada tipe timestamp dengan tingkat presisi dari **0 hingga 6 digit**. Artinya, Anda bisa mengontrol seberapa detail informasi waktu yang ingin disimpan, mulai dari hanya detik saja hingga mikrodetik.

Presisi ini sangat berguna, tetapi juga bisa menjadi jebakan jika tidak memahami bagaimana PostgreSQL memperlakukan pembulatan nilainya.

#### Konsep Dasar Precision pada Timestamp

Secara umum, sintaks penentuan presisi ditulis seperti ini:

```sql
TIMESTAMP(precision)
```

Nilai `precision` menentukan **jumlah digit di belakang koma detik**.

Contoh definisi tabel dengan berbagai tingkat presisi:

```sql
CREATE TABLE precise_events (
    no_precision TIMESTAMPTZ,           -- default: mengikuti literal (maksimal 6 digit)
    three_digits TIMESTAMPTZ(3),        -- milliseconds, contoh: 0.123
    zero_digits  TIMESTAMPTZ(0),        -- tanpa fractional seconds
    six_digits   TIMESTAMPTZ(6)         -- microseconds, contoh: 0.123456
);
```

Penjelasannya:

- Tanpa menuliskan `(precision)`, PostgreSQL akan menyimpan hingga **6 digit** jika tersedia.
- `TIMESTAMPTZ(3)` menyimpan waktu hingga **milidetik**.
- `TIMESTAMPTZ(0)` hanya menyimpan **detik**, tanpa angka desimal.
- `TIMESTAMPTZ(6)` menyimpan hingga **mikrodetik**, presisi tertinggi yang didukung.

#### Contoh Perbedaan Hasil Berdasarkan Precision

Mari lihat hasil nyata saat menggunakan fungsi `now()`:

```sql
SELECT
    now()::TIMESTAMPTZ,         -- 2024-01-15 13:01:01.923456-05
    now()::TIMESTAMPTZ(3),      -- 2024-01-15 13:01:01.923-05
    now()::TIMESTAMPTZ(0);      -- 2024-01-15 13:01:02-05 (dibulatkan!)
```

Dari hasil di atas terlihat bahwa:

- Tanpa presisi eksplisit, PostgreSQL mempertahankan detail mikrodetik.
- Presisi `(3)` memangkas detail hingga milidetik.
- Presisi `(0)` **tidak sekadar membuang desimal**, tetapi melakukan **pembulatan**.

Inilah bagian yang sering tidak disadari.

#### ⚠️ Rounding vs Truncating

Saat Anda menggunakan `TIMESTAMPTZ(n)`, PostgreSQL menerapkan **rounding (pembulatan standar)**, bukan truncating (memotong ke bawah).

Aturan rounding yang digunakan:

- Nilai **≥ 0.5** dibulatkan ke atas
- Nilai **< 0.5** dibulatkan ke bawah

Contoh konkret:

```sql
SELECT
    '13:01:01.499999'::TIMESTAMPTZ(0) AS rounded_down,  -- 13:01:01
    '13:01:01.500000'::TIMESTAMPTZ(0) AS rounded_up,    -- 13:01:02
    '13:01:01.923456'::TIMESTAMPTZ(0) AS rounded_up2;   -- 13:01:02
```

Pada contoh di atas, semua nilai fractional seconds **≥ 0.5 detik** menyebabkan detik bertambah satu.

Bandingkan dengan `date_trunc`, yang selalu melakukan **truncating**:

```sql
SELECT
    now() AS original,                         -- 13:01:01.923456
    now()::TIMESTAMPTZ(0) AS rounded,          -- 13:01:02 (hasil rounding)
    date_trunc('second', now()) AS truncated;  -- 13:01:01 (selalu dipotong ke bawah)
```

Perbedaan ini penting karena menghasilkan perilaku yang sangat berbeda.

#### Implikasi Praktis

Beberapa poin penting yang perlu diingat:

- `TIMESTAMPTZ(0)` menggunakan **pembulatan standar**
- Timestamp bisa berubah **ke detik berikutnya**
- Jika membutuhkan hasil yang **selalu konsisten ke bawah**, gunakan `date_trunc`
- **Penting:** Rounding dapat menghasilkan timestamp yang secara teknis berada **di masa depan**

#### Kapan Ini Menjadi Masalah

Kasus yang sering bermasalah adalah **audit log, event log, atau pencatatan aktivitas**.

```sql
INSERT INTO audit_log (action_time)
VALUES (now()::TIMESTAMPTZ(0));
```

Kemungkinan hasil:

- Jika `now()` = `10:30:59.6` → tersimpan `10:31:00` (waktu di masa depan)
- Jika `now()` = `10:30:59.4` → tersimpan `10:30:59` (waktu di masa lalu)

Ketidakkonsistenan ini bisa berbahaya untuk sistem audit.

Solusi yang lebih aman:

```sql
INSERT INTO audit_log (action_time)
VALUES (date_trunc('second', now()));
```

Dengan pendekatan ini:

- Waktu **selalu dipotong ke bawah**
- Tidak ada risiko timestamp melompat ke masa depan
- Lebih aman dan konsisten untuk kebutuhan logging dan audit

### c. Format Input: ISO 8601 (RECOMMENDED!)

Saat bekerja dengan timestamp, salah satu sumber bug paling umum adalah **format input yang ambigu**. Untuk menghindari hal ini, PostgreSQL (dan hampir semua sistem modern) sangat merekomendasikan penggunaan **ISO 8601**, yang sering disebut sebagai _gold standard_ untuk representasi tanggal dan waktu.

ISO 8601 dirancang agar **jelas, konsisten, dan dapat dipahami baik oleh manusia maupun mesin**, tanpa perlu menebak-nebak formatnya.

#### Format Dasar ISO 8601

Bentuk umum format ISO 8601 adalah:

```
YYYY-MM-DD HH:MI:SS[.ffffff][timezone]
```

Maknanya:

- `YYYY` → tahun (4 digit)
- `MM` → bulan
- `DD` → hari
- `HH:MI:SS` → jam, menit, detik (format 24 jam)
- `.ffffff` → fractional seconds (opsional, hingga mikrodetik)
- `timezone` → informasi zona waktu (opsional, tetapi sangat disarankan)

Dengan struktur ini, urutan tanggal selalu konsisten: dari **paling besar ke paling kecil** (tahun → bulan → hari → waktu).

#### Variasi ISO 8601 yang Valid

ISO 8601 cukup fleksibel dan PostgreSQL mendukung berbagai variasinya.

Format dengan pemisah `T` (standar formal ISO):

```sql
'2024-01-31T13:45:30'
```

Format dengan spasi sebagai pemisah (lebih nyaman dibaca manusia):

```sql
'2024-01-31 13:45:30'
```

Keduanya setara dan dipahami dengan benar oleh PostgreSQL.

Jika membutuhkan presisi lebih tinggi, Anda bisa menambahkan fractional seconds:

```sql
'2024-01-31 13:45:30.123456'
```

Ini menyimpan waktu hingga mikrodetik, sesuai kemampuan maksimal PostgreSQL.

#### Menyertakan Informasi Time Zone

Agar timestamp benar-benar tidak ambigu, ISO 8601 memungkinkan penulisan time zone secara eksplisit.

Menggunakan notasi **Zulu (`Z`)**, yang berarti UTC:

```sql
'2024-01-31 13:45:30Z'
```

Atau menggunakan **UTC offset**:

```sql
'2024-01-31 13:45:30-06:00'  -- Central Time
'2024-01-31 13:45:30+05:30'  -- India (offset setengah jam)
'2024-01-31 13:45:30+00:00'  -- UTC
```

Dengan format ini, PostgreSQL tahu persis **waktu absolut** yang Anda maksud, lalu akan melakukan konversi internal (misalnya ke UTC) jika menggunakan `TIMESTAMPTZ`.

#### Mengapa ISO 8601 Sangat Penting

Ada beberapa alasan kuat mengapa ISO 8601 hampir selalu menjadi pilihan terbaik:

- ✅ **Tidak ambigu**
  Tidak ada kebingungan antara format seperti `01-02-2024` (apakah 1 Februari atau 2 Januari).

- ✅ **Diakui secara internasional**
  Digunakan secara konsisten di database, API, log sistem, dan standar industri.

- ✅ **Sortable secara natural**
  Jika disortir sebagai string, urutannya tetap benar karena formatnya dari besar ke kecil.

- ✅ **Machine-readable**
  Mudah diparse oleh bahasa pemrograman, database, dan sistem lain tanpa konfigurasi tambahan.

Dengan menggunakan ISO 8601 secara konsisten, Anda mengurangi risiko kesalahan interpretasi waktu, membuat sistem lebih stabil, dan mempermudah integrasi antar layanan.

### d. Ambiguous Date Formats (AVOID!)

Salah satu sumber kesalahan paling berbahaya dalam sistem adalah **format tanggal yang ambigu**. Masalahnya bukan karena PostgreSQL tidak mampu membacanya, tetapi karena **manusia dan sistem dari wilayah berbeda menafsirkannya secara berbeda**. Inilah alasan kuat mengapa format tanggal tertentu sebaiknya **benar-benar dihindari**.

#### Contoh Format yang Ambigu

Perhatikan dua contoh sederhana berikut:

```sql
'1/3/2024'
```

Tanggal ini bisa berarti:

- **USA (MDY)** → January 3, 2024
- **Sebagian besar dunia (DMY)** → March 1, 2024

Contoh lain:

```sql
'3/1/2024'
```

Maknanya justru terbalik:

- **USA (MDY)** → March 1, 2024
- **Sebagian besar dunia (DMY)** → January 3, 2024

Tanpa konteks tambahan, format seperti ini **tidak pernah aman**, karena interpretasinya bergantung pada asumsi regional.

#### Bagaimana PostgreSQL Menangani Ambiguitas

PostgreSQL mencoba mengatasi ambiguitas ini melalui konfigurasi bernama `datestyle`.

```sql
SHOW datestyle;
```

Contoh hasil:

```
ISO, DMY
```

Arti masing-masing bagian:

- Bagian pertama (`ISO`) menentukan **format output**
- Bagian kedua (`DMY`, `MDY`, atau `YMD`) menentukan **cara PostgreSQL menafsirkan tanggal ambigu**

Dengan kata lain, input seperti `'1/3/2024'` akan ditafsirkan **berdasarkan setting ini**, bukan berdasarkan niat Anda saat menulis query.

#### Contoh Perubahan Interpretasi dengan `datestyle`

Menggunakan gaya Amerika (bulan-hari-tahun):

```sql
SET datestyle = 'ISO, MDY';
SELECT '1/3/2024'::DATE;
```

Hasilnya:

```
2024-01-03
```

Artinya: **January 3, 2024**.

Menggunakan gaya internasional (hari-bulan-tahun):

```sql
SET datestyle = 'ISO, DMY';
SELECT '1/3/2024'::DATE;
```

Hasilnya:

```
2024-03-01
```

Artinya: **March 1, 2024**.

Query yang sama, literal yang sama, tetapi hasilnya **berbeda total** hanya karena konfigurasi.

#### Perbedaan Format Output `datestyle`

Selain memengaruhi cara membaca input, `datestyle` juga menentukan **bagaimana tanggal ditampilkan**.

Format ISO (direkomendasikan):

```sql
SET datestyle = 'ISO, DMY';
SELECT '2024-01-31'::DATE;
```

Output:

```
2024-01-31
```

Format ini jelas, konsisten, dan tidak ambigu.

Format SQL (nama yang menyesatkan karena bukan standar SQL resmi):

```sql
SET datestyle = 'SQL, DMY';
SELECT '2024-01-31'::DATE;
```

Output:

```
01/31/2024
```

Format German:

```sql
SET datestyle = 'German, DMY';
SELECT '2024-01-31'::DATE;
```

Output:

```
31.01.2024
```

Format European (mencakup German):

```sql
SET datestyle = 'European';
SELECT '2024-01-31'::DATE;
```

Output:

```
31.01.2024
```

Perlu ditekankan bahwa **format output tidak mengubah nilai tanggal**, tetapi dapat menambah kebingungan jika digunakan untuk pertukaran data antar sistem.

#### Best Practice yang Harus Diikuti

```sql
-- ❌ JANGAN LAKUKAN INI
-- Bergantung pada datestyle dan regional setting
INSERT INTO events (event_date) VALUES ('1/3/2024');
```

Pendekatan ini berbahaya karena hasilnya bisa berubah tergantung server, user, atau environment.

Sebaliknya, gunakan format yang eksplisit dan tidak ambigu:

```sql
-- ✅ LAKUKAN INI
INSERT INTO events (event_date) VALUES ('2024-01-03');
```

Format ISO `YYYY-MM-DD` selalu memiliki satu arti yang jelas, tidak bergantung pada `datestyle`, dan aman digunakan di mana pun.

### e. Unix Timestamp Conversion

Unix timestamp adalah representasi waktu dalam bentuk **jumlah detik sejak epoch**, yaitu **1970-01-01 00:00:00 UTC**. Format ini sangat umum digunakan di API, sistem logging, message queue, dan berbagai sistem lintas platform karena sederhana, konsisten, dan tidak terikat pada time zone lokal.

PostgreSQL menyediakan fungsi bawaan untuk melakukan konversi dua arah antara **Unix timestamp** dan **timestamp PostgreSQL**.

#### Konversi Unix Timestamp ke Timestamp PostgreSQL

Untuk mengonversi Unix timestamp ke tipe timestamp PostgreSQL, gunakan fungsi `to_timestamp()`.

```sql
SELECT to_timestamp(1705329690);
```

Hasilnya:

```
2024-01-15 12:34:50+00
```

Artinya, nilai `1705329690` detik sejak epoch diterjemahkan menjadi waktu **15 Januari 2024 pukul 12:34:50 UTC**.

Jika Unix timestamp memiliki fractional seconds (biasanya dalam bentuk float), PostgreSQL akan mempertahankan presisinya:

```sql
SELECT to_timestamp(1705329690.123456);
```

Hasilnya:

```
2024-01-15 12:34:50.123456+00
```

Mikrodetik tetap disimpan dengan akurat, selama berada dalam batas presisi PostgreSQL.

Untuk memastikan tipe data hasilnya:

```sql
SELECT pg_typeof(to_timestamp(1705329690));
```

Hasil:

```
timestamp with time zone
```

Ini penting untuk diketahui karena akan memengaruhi perilaku konversi time zone saat data dibaca oleh client.

#### Karakteristik Penting `to_timestamp()`

Fungsi `to_timestamp()` memiliki beberapa sifat penting yang membuatnya aman digunakan:

- Selalu mengembalikan **`TIMESTAMPTZ` (timestamp with time zone)**
- Nilainya selalu direpresentasikan dalam **UTC (+00:00)**
- Fractional seconds tetap dipertahankan jika tersedia
- Secara konseptual benar, karena Unix timestamp **selalu berbasis UTC**

Karena Unix timestamp tidak mengandung informasi zona waktu, pendekatan ini konsisten dan bebas ambiguitas.

#### Konversi Timestamp ke Unix Timestamp

Sebaliknya, untuk mengubah timestamp PostgreSQL menjadi Unix timestamp, gunakan fungsi `EXTRACT(EPOCH FROM ...)`.

```sql
SELECT EXTRACT(EPOCH FROM now());
```

Contoh hasil:

```
1705329690.123456
```

Nilai ini berupa **floating-point number** yang mencakup fractional seconds.

Jika Anda membutuhkan nilai integer (misalnya untuk API atau sistem yang hanya menerima detik):

```sql
SELECT EXTRACT(EPOCH FROM now())::BIGINT;
```

Hasilnya:

```
1705329690
```

Dengan melakukan casting ke `BIGINT`, fractional seconds dibuang, dan Anda mendapatkan Unix timestamp dalam bentuk **detik bulat**.

Konversi dua arah ini membuat PostgreSQL sangat cocok untuk berintegrasi dengan sistem lain yang menggunakan Unix timestamp sebagai format waktu utama.

### f. Date vs Time vs Timestamp

PostgreSQL menyediakan **tipe data yang berbeda untuk merepresentasikan konsep waktu yang berbeda pula**. Memisahkan konsep ini penting agar data yang disimpan benar secara makna, mudah diolah, dan tidak menimbulkan kebingungan di kemudian hari.

Alih-alih menggunakan satu tipe untuk semua kasus, PostgreSQL memaksa kita berpikir: _apakah ini hanya tanggal, hanya jam, atau momen waktu yang lengkap?_

#### Gambaran Umum Tipe Waktu di PostgreSQL

Berikut contoh tabel yang memperlihatkan perbedaan tiap tipe:

```sql
CREATE TABLE time_types (
    -- Hanya tanggal (tanpa waktu)
    birth_date DATE,                -- 2024-01-15

    -- Hanya waktu (tanpa tanggal)
    opening_time TIME,              -- 09:00:00

    -- Tanggal + waktu lengkap (direkomendasikan)
    created_at TIMESTAMPTZ,         -- 2024-01-15 09:00:00+00

    -- Interval atau durasi waktu
    duration INTERVAL               -- '2 hours 30 minutes'
);
```

Masing-masing kolom di atas merepresentasikan **jenis informasi waktu yang berbeda**, bukan sekadar format yang berbeda.

#### DATE — Hanya Tanggal

Tipe `DATE` menyimpan **tanggal saja**, tanpa informasi jam, menit, atau detik.

Contoh penggunaan yang tepat:

- Tanggal lahir
- Tanggal deadline
- Tanggal acara atau hari libur

Karena tidak ada unsur waktu, `DATE` **tidak terpengaruh time zone** dan relatif aman digunakan untuk data berbasis kalender.

Contoh nilai:

```
2024-01-15
```

#### TIME — Hanya Waktu (Tanpa Tanggal)

Tipe `TIME` menyimpan **jam, menit, dan detik**, tetapi **tidak memiliki konteks tanggal**.

Contoh nilai:

```
09:00:00
```

Secara praktis, tipe ini **jarang berguna** karena:

- Tidak tahu jam ini terjadi di hari apa
- Sulit digunakan untuk perhitungan lintas hari
- Tidak cocok untuk event nyata di dunia

Kasus yang masih masuk akal adalah seperti **jam buka toko** atau **jadwal harian**, tetapi bahkan dalam banyak kasus tersebut, `TIMESTAMPTZ` tetap lebih fleksibel.

#### TIMESTAMPTZ — Tanggal dan Waktu Lengkap

`TIMESTAMPTZ` (timestamp with time zone) adalah tipe yang paling sering dan **paling direkomendasikan** untuk merepresentasikan momen waktu yang nyata.

Contoh nilai:

```
2024-01-15 09:00:00+00
```

Kapan menggunakan `TIMESTAMPTZ`:

- `created_at`
- `updated_at`
- Waktu kejadian (event_time)
- Log dan audit trail

Tipe ini:

- Disimpan secara konsisten dalam UTC
- Aman untuk aplikasi lintas time zone
- Mendukung perhitungan waktu dengan benar

Jika Anda ragu harus memilih tipe apa, **TIMESTAMPTZ hampir selalu pilihan yang benar**.

#### INTERVAL — Durasi atau Selisih Waktu

Berbeda dari tipe lainnya, `INTERVAL` **tidak merepresentasikan titik waktu**, melainkan **durasi atau jarak waktu**.

Contoh nilai:

```
'2 hours 30 minutes'
```

Contoh penggunaan:

- Lama proses
- Masa berlaku
- Selisih antara dua timestamp
- Timeout atau SLA

`INTERVAL` sering digunakan bersama `TIMESTAMPTZ` untuk operasi seperti:

```sql
SELECT now() + INTERVAL '1 day 2 hours';
```

#### Ringkasan Kapan Menggunakan Masing-Masing

- **DATE** → Saat Anda hanya peduli pada tanggal
- **TIME** → Sangat jarang, hanya untuk jam tanpa konteks hari
- **TIMESTAMPTZ** → Untuk semua momen waktu yang nyata dan penting
- **INTERVAL** → Untuk durasi atau selisih waktu, bukan tanggal kejadian

Dengan memilih tipe yang tepat sejak awal, Anda menghindari banyak masalah logika dan konversi di masa depan.

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
