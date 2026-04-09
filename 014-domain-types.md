# 📝 Domain Types di PostgreSQL (VERSI TERKOREKSI)

## 1. Ringkasan Singkat

Video ini membahas Domain Types di PostgreSQL, sebuah fitur yang mengenkapsulasi tipe data dan check constraints menjadi satu custom type yang dapat digunakan kembali (reusable). Domain memungkinkan kita membuat "tipe data custom" yang sebenarnya adalah wrapper dari tipe data existing ditambah validasi. Ini sangat berguna untuk konsistensi validasi di banyak kolom atau tabel, dan lebih mudah dimodifikasi dibanding check constraints biasa.

## 2. Konsep Utama

### a. Apa itu Domain?

Domain adalah **pembungkus (wrapper)** yang menggabungkan _tipe data dasar_ dengan satu atau lebih **check constraint**, lalu diberi **nama khusus** sehingga dapat digunakan kembali sebagai _custom type_. Dengan domain, aturan validasi data tidak lagi tersebar di banyak tabel, tetapi didefinisikan sekali dan dipakai secara konsisten di mana pun dibutuhkan.

#### Konsep Dasar Domain

Tanpa domain, setiap kolom yang memiliki aturan validasi serupa harus menuliskan constraint yang sama berulang-ulang. Hal ini membuat skema database menjadi lebih panjang, rawan inkonsistensi, dan sulit dirawat.

```sql
-- Tanpa domain: Harus tulis constraint berulang-ulang
CREATE TABLE orders (
    price NUMERIC CHECK (price > 0),
    discount_price NUMERIC CHECK (discount_price > 0)  -- Duplikasi constraint!
);

CREATE TABLE invoices (
    amount NUMERIC CHECK (amount > 0)  -- Duplikasi lagi!
);
```

Pada contoh di atas, aturan _“nilai harus lebih besar dari 0”_ ditulis di setiap kolom. Jika suatu saat aturan ini berubah (misalnya nilai minimum menjadi 1), maka semua definisi kolom tersebut harus diperbarui satu per satu.

Dengan domain, kita mendefinisikan aturan tersebut **sekali saja**, lalu menggunakannya kembali di berbagai tabel.

```sql
-- Dengan domain: Define sekali, pakai berkali-kali
CREATE DOMAIN positive_price AS NUMERIC CHECK (VALUE > 0);

CREATE TABLE orders (
    price positive_price,
    discount_price positive_price  -- Reuse!
);

CREATE TABLE invoices (
    amount positive_price  -- Reuse lagi!
);
```

Di sini, `positive_price` adalah domain yang membungkus tipe `NUMERIC` dengan constraint `VALUE > 0`. Setiap kolom yang menggunakan domain ini otomatis mewarisi aturan tersebut tanpa perlu menuliskannya ulang.

#### Struktur atau Formula Domain

Secara konseptual, sebuah domain dapat diringkas dengan formula berikut:

```
Domain = Base Type + Check Constraint(s) + Custom Name
```

Artinya, domain selalu memiliki:

- **Base Type** → tipe data asli seperti `NUMERIC`, `VARCHAR`, `DATE`, dan sebagainya
- **Check Constraint(s)** → aturan validasi yang harus dipenuhi nilainya
- **Custom Name** → nama domain yang akan digunakan sebagai tipe data baru

#### Domain Berbasis Domain Lain

Perlu diluruskan bahwa domain **boleh** dibuat berdasarkan domain lain. Ini berarti kita dapat membangun domain yang lebih spesifik di atas domain yang sudah ada, sehingga aturan validasi menjadi bertingkat dan semakin terstruktur.

```sql
-- Base domain
CREATE DOMAIN positive_number AS NUMERIC CHECK (VALUE > 0);

-- Domain based on another domain (derived domain)
CREATE DOMAIN price AS positive_number CHECK (VALUE < 1000000);

-- This is VALID in PostgreSQL!
```

Pada contoh ini:

- `positive_number` memastikan nilai selalu lebih besar dari 0
- `price` mewarisi aturan tersebut, lalu menambahkan constraint baru agar nilainya juga kurang dari 1.000.000

Pendekatan ini sangat berguna untuk membangun hierarki aturan bisnis yang jelas dan reusable, misalnya membedakan antara _angka positif umum_ dan _harga dengan batas maksimum tertentu_, tanpa perlu menduplikasi logika validasi.

### b. Syntax Membuat Domain

Membuat domain pada dasarnya adalah proses mendefinisikan _tipe data baru_ yang memiliki aturan validasi bawaan. Secara sintaks, PostgreSQL menyediakan perintah `CREATE DOMAIN` untuk menggabungkan tipe data dasar dengan satu atau lebih constraint, sehingga setiap nilai yang menggunakan domain tersebut otomatis harus memenuhi aturan yang telah ditentukan.

#### Bentuk Umum Syntax

```sql
CREATE DOMAIN domain_name AS base_type
    [CONSTRAINT constraint_name]
    CHECK (condition_using_VALUE);
```

Syntax di atas menunjukkan bahwa sebuah domain selalu didefinisikan di atas _base type_ tertentu, lalu diperkuat dengan aturan validasi menggunakan `CHECK`.

#### Penjelasan Setiap Komponen

- `domain_name`
  Nama domain yang akan dibuat. Setelah didefinisikan, nama ini dapat digunakan seperti tipe data bawaan (`INTEGER`, `TEXT`, dan sebagainya) pada kolom tabel.

- `base_type`
  Tipe data dasar yang menjadi fondasi domain, misalnya `TEXT`, `INTEGER`, atau `NUMERIC`. Penting untuk dicatat bahwa _base type ini juga bisa berupa domain lain_, sehingga domain dapat disusun secara bertingkat.

- `VALUE`
  Keyword khusus di dalam domain yang merepresentasikan nilai yang sedang divalidasi. Berbeda dengan constraint pada tabel yang mereferensikan nama kolom, di dalam domain kita selalu menggunakan `VALUE`.

- `CHECK`
  Constraint yang berisi kondisi logis. Setiap nilai yang dimasukkan ke kolom bertipe domain harus memenuhi kondisi ini, jika tidak maka operasi insert atau update akan gagal.

#### Contoh Penerapan Domain

Berikut beberapa contoh domain sederhana yang sering ditemui dalam kebutuhan nyata.

```sql
-- Domain untuk email
CREATE DOMAIN email AS TEXT
    CONSTRAINT email_format
    CHECK (VALUE LIKE '%@%');
```

Domain `email` membungkus tipe `TEXT` dan memastikan bahwa nilai yang dimasukkan mengandung karakter `@`. Setiap kolom bertipe `email` otomatis akan menolak data yang tidak memenuhi pola ini.

```sql
-- Domain untuk persentase
CREATE DOMAIN percentage AS NUMERIC
    CONSTRAINT valid_percentage
    CHECK (VALUE BETWEEN 0 AND 100);
```

Domain `percentage` memastikan bahwa nilai numerik selalu berada di rentang 0 hingga 100. Ini sangat berguna untuk kolom seperti diskon atau progress, sehingga tidak mungkin menyimpan nilai di luar batas logis tersebut.

```sql
-- Domain untuk umur
CREATE DOMAIN age AS INTEGER
    CONSTRAINT valid_age
    CHECK (VALUE >= 0 AND VALUE <= 150);
```

Domain `age` membatasi nilai umur agar tidak negatif dan tidak melebihi batas yang masuk akal. Dengan cara ini, aturan validasi umur tidak perlu diulang di setiap tabel yang memiliki kolom umur.

Melalui contoh-contoh ini, terlihat bahwa domain membantu memindahkan logika validasi ke level tipe data. Hasilnya, struktur database menjadi lebih rapi, konsisten, dan mudah dipelihara karena aturan yang sama tidak perlu ditulis ulang di banyak tempat.

### c. Studi Kasus: US Postal Code

Pada studi kasus ini, kita melihat contoh nyata bagaimana domain membantu memvalidasi format data yang tidak bisa ditangani dengan baik oleh tipe data numerik biasa. Contohnya adalah **US Postal Code (ZIP Code)**, yang memiliki dua format resmi.

#### Format US Postal Code

US Postal Code mengenal dua bentuk penulisan:

- Format pendek: **5 digit**, misalnya `12345`
- Format panjang: **5 digit + tanda minus + 4 digit**, misalnya `12345-6789`

Kedua format ini sama-sama valid dan sering digunakan dalam sistem alamat.

#### Masalah Jika Menggunakan Tipe Numerik

```sql
-- Tidak bisa pakai INTEGER/NUMERIC karena:
-- 1. Bisa kehilangan leading zero (07432 jadi 7432)
-- 2. Tidak support dash (12345-6789)

-- Harus pakai TEXT, tapi butuh validasi format
```

Jika kita menggunakan `INTEGER` atau `NUMERIC`, ada dua masalah serius:

1. **Leading zero akan hilang**. Kode pos seperti `07432` akan disimpan sebagai `7432`, yang jelas mengubah makna data.
2. **Format dengan dash tidak didukung**, karena angka murni tidak bisa menyimpan karakter `-`.

Menggunakan `TEXT` memang menyelesaikan masalah penyimpanan, tetapi membuka risiko baru: **tidak ada jaminan format data yang masuk benar**. Tanpa validasi, nilai seperti `123`, `ABC`, atau `12345-XYZ` bisa saja tersimpan.

#### Solusi Menggunakan Domain

Domain memberikan solusi yang rapi dengan menggabungkan `TEXT` dan aturan validasi format dalam satu definisi reusable.

```sql
CREATE DOMAIN us_postal_code AS TEXT
    CONSTRAINT format
    CHECK (
        VALUE ~ '^\d{5}$'           -- Format: 5 digit
        OR
        VALUE ~ '^\d{5}-\d{4}$'     -- Format: 5 digit - 4 digit
    );
```

Domain `us_postal_code` memastikan bahwa setiap nilai:

- Entah **tepat 5 digit**, atau
- **5 digit diikuti dash dan 4 digit**

Validasi dilakukan menggunakan regular expression (regex) PostgreSQL. Dengan cara ini, aturan format berada di level tipe data, bukan tersebar di setiap tabel.

Domain ini kemudian digunakan langsung pada definisi tabel.

```sql
-- Sekarang bisa dipakai di tabel
CREATE TABLE addresses (
    street TEXT,
    city TEXT,
    postal us_postal_code NOT NULL  -- Domain + NOT NULL di KOLOM (best practice!)
);
```

Perlu diperhatikan bahwa `NOT NULL` diletakkan di level kolom, bukan di domain. Ini adalah _best practice_, karena domain fokus pada **validasi format dan nilai**, sementara aturan _mandatory/optional_ ditentukan oleh kebutuhan masing-masing tabel.

#### Penjelasan Regex yang Digunakan

Agar lebih mudah dipahami, berikut arti dari setiap bagian regex:

- `^`
  Menandai awal string. Memastikan tidak ada karakter sebelum pola yang diuji.

- `\d{5}`
  Tepat 5 digit angka.

- `-`
  Karakter dash literal.

- `\d{4}`
  Tepat 4 digit angka.

- `$`
  Menandai akhir string. Memastikan tidak ada karakter tambahan setelah pola.

- `~`
  Operator PostgreSQL untuk mencocokkan nilai dengan regular expression.

Dengan kombinasi `^` dan `$`, kita memastikan bahwa **seluruh isi string** harus sesuai pola, bukan hanya sebagian.

#### Pengujian Domain

Setelah domain dan tabel dibuat, kita bisa langsung menguji perilakunya dengan `INSERT`.

```sql
-- Test valid values
INSERT INTO addresses VALUES ('Main St', 'Dallas', '07432');      -- ✅ OK
INSERT INTO addresses VALUES ('Oak Ave', 'Boston', '12345-6789'); -- ✅ OK
```

Kedua contoh di atas berhasil disimpan karena formatnya sesuai dengan aturan domain, termasuk kasus `07432` yang tetap mempertahankan leading zero.

```sql
-- Test invalid values
INSERT INTO addresses VALUES ('Elm St', 'NYC', '7432');           -- ❌ ERROR: format
-- ERROR: value for domain us_postal_code violates check constraint "format"

INSERT INTO addresses VALUES ('Pine Rd', 'LA', '123');            -- ❌ ERROR: format
INSERT INTO addresses VALUES ('Maple Dr', 'SF', '12345-ABC');     -- ❌ ERROR: format (harus digit)
```

Untuk data yang tidak sesuai, PostgreSQL langsung menolak dengan error dari domain constraint. Ini menunjukkan keuntungan utama domain: **validasi konsisten dan otomatis**, tanpa perlu logika tambahan di level aplikasi atau query yang berulang-ulang.

### d. Keyword `VALUE` dalam Domain

`VALUE` adalah keyword khusus yang **hanya digunakan dalam definisi domain**. Fungsinya adalah mereferensikan nilai yang sedang divalidasi oleh domain tersebut. Karena domain didefinisikan secara terpisah dari tabel dan kolom, PostgreSQL membutuhkan cara generik untuk menunjuk nilai yang diuji, dan di sinilah peran `VALUE`.

#### Perbedaan dengan Check Constraint pada Tabel

Perbedaan paling mencolok antara check constraint pada tabel dan constraint di dalam domain terletak pada **cara mereferensikan nilai**.

```sql
-- Check Constraint: Pakai nama kolom
CREATE TABLE products (
    price NUMERIC CHECK (price > 0)  -- Referensi nama kolom
);
```

Pada check constraint di level tabel, kita selalu menyebut **nama kolom**, karena constraint tersebut melekat langsung pada kolom tertentu.

Sebaliknya, pada domain belum ada konteks kolom sama sekali.

```sql
-- Domain: Pakai VALUE (karena belum ada nama kolom)
CREATE DOMAIN positive_amount AS NUMERIC
    CHECK (VALUE > 0);  -- Referensi VALUE
```

Domain didefinisikan sebagai objek mandiri. Ia belum tahu akan digunakan di kolom mana, sehingga tidak mungkin menyebut nama kolom secara spesifik. Oleh karena itu, PostgreSQL menyediakan keyword `VALUE` sebagai pengganti nama kolom.

#### Mengapa Harus Menggunakan `VALUE`?

Ada beberapa alasan utama mengapa `VALUE` menjadi bagian penting dalam domain:

- Domain adalah **standalone object**, tidak terikat pada satu tabel atau satu kolom tertentu
- Satu domain bisa digunakan di banyak kolom dengan **nama yang berbeda-beda**
- `VALUE` bertindak sebagai **placeholder universal** untuk nilai apa pun yang sedang disimpan dan divalidasi oleh domain

Dengan pendekatan ini, satu definisi domain dapat berlaku konsisten di berbagai konteks.

#### Contoh Penggunaan `VALUE` yang Lebih Kompleks

```sql
CREATE DOMAIN email AS TEXT
    CHECK (VALUE LIKE '%@%' AND length(VALUE) > 5);
```

Domain `email` di atas memvalidasi dua hal sekaligus:

- Nilai harus mengandung karakter `@`
- Panjang string harus lebih dari 5 karakter

Ketika domain ini digunakan di tabel, `VALUE` akan secara otomatis merujuk ke nilai kolom tempat domain tersebut dipakai.

```sql
-- Nanti bisa dipakai dengan nama kolom berbeda:
CREATE TABLE users (
    primary_email email,      -- VALUE akan = primary_email
    backup_email email        -- VALUE akan = backup_email
);
```

Pada kolom `primary_email`, `VALUE` merepresentasikan nilai `primary_email`. Pada kolom `backup_email`, `VALUE` merepresentasikan nilai `backup_email`. Meskipun nama kolom berbeda, aturan validasinya tetap sama karena berasal dari domain yang sama.

Inilah kekuatan utama `VALUE`: ia memungkinkan domain tetap **fleksibel, reusable, dan konsisten**, tanpa perlu mengetahui konteks kolom secara spesifik.

### e. Domain vs Check Constraint

Domain dan check constraint sama-sama digunakan untuk menjaga validitas data, tetapi keduanya bekerja di level yang berbeda dan memiliki karakteristik yang tidak sepenuhnya sama. Memahami perbedaan ini membantu kita memilih pendekatan yang paling tepat sesuai kebutuhan desain database.

#### Perbandingan Konseptual

| Aspek                  | Domain                                                                   | Check Constraint                                           |
| ---------------------- | ------------------------------------------------------------------------ | ---------------------------------------------------------- |
| **Reusability**        | Domain dapat digunakan berulang kali di banyak tabel dan kolom           | Check constraint harus ditulis ulang di setiap kolom       |
| **Multi-column**       | Domain hanya berlaku untuk satu kolom                                    | Check constraint bisa mereferensikan lebih dari satu kolom |
| **Modification**       | Domain dapat diubah menggunakan `ALTER DOMAIN`                           | Check constraint umumnya harus dihapus lalu dibuat ulang   |
| **Gradual rollout**    | Domain mendukung `NOT VALID`, sehingga validasi bisa diterapkan bertahap | Tidak ada mekanisme serupa                                 |
| **Encapsulation**      | Tipe data dan aturan validasi dibungkus menjadi satu kesatuan            | Tipe data dan constraint terpisah                          |
| **Komunikasi**         | Nama domain bisa merepresentasikan makna bisnis secara jelas             | Kejelasan makna bergantung pada penamaan constraint        |
| **Container types**    | Ada keterbatasan saat domain digunakan di tipe kontainer tertentu        | Tidak memiliki keterbatasan khusus                         |
| **Automatic downcast** | Nilai domain dapat diturunkan ke base type secara otomatis               | Tidak relevan                                              |

Dari tabel ini terlihat bahwa domain unggul dalam hal **konsistensi, reusability, dan kejelasan makna**, sementara check constraint lebih fleksibel ketika validasi melibatkan **lebih dari satu kolom**.

Secara praktis, **gunakan domain ketika validasi bersifat umum dan berulang**, serta ingin dikomunikasikan sebagai konsep bisnis yang jelas. Gunakan **check constraint langsung pada tabel** ketika validasi melibatkan lebih dari satu kolom atau bersifat sangat spesifik pada satu tabel saja.

### f. BATASAN: Multi-Column Constraint dengan Domain

Domain sangat kuat untuk memvalidasi **nilai individual pada satu kolom**, tetapi memiliki batasan yang penting untuk dipahami. Salah satu batasan utama tersebut adalah domain **tidak bisa digunakan untuk validasi yang melibatkan lebih dari satu kolom**.

#### Keterbatasan Domain pada Multi-Column Constraint

```sql
-- ❌ TIDAK BISA: Multi-column logic dalam domain
CREATE DOMAIN price_with_discount AS ??? -- Tidak ada cara untuk reference 2 kolom
```

Domain didefinisikan tanpa konteks tabel dan tanpa pengetahuan tentang kolom lain. Ia hanya mengenal satu hal, yaitu `VALUE`, yang merepresentasikan satu nilai saja. Karena itu, tidak ada mekanisme untuk membandingkan dua kolom atau mengekspresikan hubungan antar kolom di dalam domain.

Jika kita membutuhkan logika seperti _harga harus lebih besar dari diskon_, maka validasi tersebut **tidak mungkin** ditaruh di domain.

#### Solusi yang Tepat: Table-Level Check Constraint

Untuk validasi yang melibatkan hubungan antar kolom, PostgreSQL menyediakan check constraint di level tabel.

```sql
-- ✅ HARUS: Tetap pakai table-level check constraint
CREATE TABLE products (
    price positive_amount,           -- Domain untuk single column
    discount positive_amount,        -- Domain untuk single column
    CHECK (price > discount)         -- Table constraint untuk relationship
);
```

Pada contoh ini:

- Domain `positive_amount` memastikan bahwa `price` dan `discount` masing-masing bernilai positif
- Check constraint di level tabel memastikan **relasi antar kolom**, yaitu `price` harus lebih besar dari `discount`

Pendekatan ini memisahkan tanggung jawab dengan jelas: domain untuk validasi nilai individual, dan table constraint untuk aturan relasional.

#### Best Practice: Kombinasi Domain dan Check Constraint

Pendekatan terbaik dalam desain skema adalah **mengombinasikan domain dan check constraint**, bukan memilih salah satu secara eksklusif.

```sql
-- Domain untuk validasi individual
CREATE DOMAIN positive_amount AS NUMERIC CHECK (VALUE > 0);
CREATE DOMAIN email AS TEXT CHECK (VALUE LIKE '%@%');

-- Pakai domain + table constraint
CREATE TABLE users (
    primary_email email,
    backup_email email,
    balance positive_amount,
    credit_limit positive_amount,

    -- Table constraint untuk relationship antar kolom
    CHECK (primary_email != backup_email),
    CHECK (balance <= credit_limit)
);
```

Dalam contoh ini:

- Domain `email` memastikan setiap kolom email memiliki format dasar yang benar
- Domain `positive_amount` memastikan nilai numerik selalu positif
- Check constraint di level tabel menangani aturan yang **melibatkan lebih dari satu kolom**, seperti memastikan email utama dan cadangan tidak sama, atau saldo tidak melebihi batas kredit

Dengan pola ini, skema database menjadi lebih bersih, logis, dan mudah dipelihara. Setiap jenis validasi ditempatkan di level yang tepat sesuai tanggung jawabnya, sehingga aturan bisnis lebih jelas dan tidak tercampur satu sama lain.

### g. Memodifikasi Domain dengan ALTER

Salah satu keunggulan penting dari domain dibandingkan check constraint biasa adalah **domain dapat dimodifikasi secara langsung menggunakan `ALTER DOMAIN`**. Ini membuat domain jauh lebih fleksibel dan ramah terhadap perubahan kebutuhan bisnis, terutama ketika skema sudah berisi data dalam jumlah besar.

#### ALTER Domain Dasar

Berikut beberapa operasi umum yang bisa dilakukan pada domain.

```sql
-- Membuat domain
CREATE DOMAIN age AS INTEGER CHECK (VALUE >= 0);
```

Domain `age` di atas memastikan bahwa nilai umur tidak negatif.

```sql
-- Menambah constraint baru
ALTER DOMAIN age ADD CONSTRAINT max_age CHECK (VALUE <= 150);
```

Perintah ini menambahkan aturan baru tanpa perlu menghapus domain atau mengubah tabel yang menggunakannya. Setelah ini, setiap kolom bertipe `age` otomatis memiliki dua aturan: minimal 0 dan maksimal 150.

```sql
-- Menghapus constraint
ALTER DOMAIN age DROP CONSTRAINT max_age;
```

Jika aturan tersebut tidak lagi relevan, constraint dapat dihapus dengan aman.

```sql
-- Rename domain
ALTER DOMAIN age RENAME TO person_age;
```

Rename domain berguna ketika ingin memperjelas makna bisnis tanpa harus mengubah struktur tabel yang sudah ada.

#### NOT VALID: Gradual Rollout Constraint

Salah satu fitur paling powerful dari domain adalah kemampuan menambahkan constraint baru secara **bertahap** menggunakan `NOT VALID`. Fitur ini sangat membantu saat melakukan migrasi pada sistem yang sudah memiliki data lama.

```sql
-- Scenario: Domain existing dengan data lama
CREATE DOMAIN us_postal_code AS TEXT
    CHECK (VALUE ~ '^\d{5}$');
```

Awalnya, domain `us_postal_code` hanya mendukung format 5 digit.

```sql
-- Table dengan data lama (sudah ada data yang tidak valid)
CREATE TABLE addresses (
    postal us_postal_code
);
```

Pada titik ini, mungkin sudah ada data lama dengan format lain, misalnya `12345-6789`, yang belum didukung oleh domain.

```sql
-- ALTER dengan NOT VALID: Tidak check data existing
ALTER DOMAIN us_postal_code
    ADD CONSTRAINT extended_format
    CHECK (VALUE ~ '^\d{5}$' OR VALUE ~ '^\d{5}-\d{4}$')
    NOT VALID;
```

Dengan `NOT VALID`, PostgreSQL:

- **Tidak langsung memeriksa data yang sudah ada**
- Tetapi **langsung memvalidasi data baru** yang masuk setelah perubahan ini

Artinya, kita bisa mulai menerima format baru tanpa harus langsung membersihkan semua data lama.

```sql
-- Insert baru akan divalidasi
INSERT INTO addresses VALUES ('12345-6789');  -- ✅ OK (new rule)
INSERT INTO addresses VALUES ('123');         -- ❌ ERROR (new rule)
```

Data baru wajib mengikuti aturan baru, sementara data lama yang belum sesuai masih dibiarkan untuk sementara.

```sql
-- Update data lama satu per satu
UPDATE addresses SET postal = '12345' WHERE postal = 'invalid_old_data';
```

Setelah semua data lama diperbaiki, barulah constraint ditegakkan sepenuhnya.

```sql
-- Setelah semua data valid, enforce fully
ALTER DOMAIN us_postal_code VALIDATE CONSTRAINT extended_format;
```

Perintah `VALIDATE CONSTRAINT` akan memeriksa seluruh data yang ada dan memastikan semuanya sudah sesuai aturan.

#### Risiko Race Condition yang Perlu Dipahami

Perlu diperhatikan bahwa `ALTER DOMAIN ADD CONSTRAINT` dengan `NOT VALID` **tidak sepenuhnya bulletproof**.

Dokumentasi resmi PostgreSQL memperingatkan bahwa:

- Perintah ini tidak bisa “melihat” baris yang sedang di-insert atau di-update dalam transaksi lain yang belum di-commit
- Jika ada operasi paralel, ada kemungkinan data tidak valid sempat masuk

Karena itu, penggunaan fitur ini harus disertai prosedur yang aman.

#### Safe Workflow untuk NOT VALID

```sql
-- 1. Add constraint dengan NOT VALID
ALTER DOMAIN us_postal_code
    ADD CONSTRAINT extended_format
    CHECK (VALUE ~ '^\d{5}$' OR VALUE ~ '^\d{5}-\d{4}$')
    NOT VALID;

-- 2. COMMIT
COMMIT;

-- 3. TUNGGU semua transactions sebelum commit selesai
-- (Monitor dengan pg_stat_activity)

-- 4. VALIDATE constraint (akan check semua data existing)
ALTER DOMAIN us_postal_code VALIDATE CONSTRAINT extended_format;
```

Workflow ini meminimalkan risiko data tidak valid masuk selama masa transisi.

#### Alur Migrasi Bertahap

Secara konseptual, proses migrasi bertahap dapat diringkas sebagai berikut:

```
1. ALTER DOMAIN dengan NOT VALID
   ↓
2. COMMIT transaction
   ↓
3. Tunggu old transactions finish
   ↓
4. Data baru: Langsung divalidasi dengan rule baru
   Data lama: Tidak di-touch, masih boleh invalid
   ↓
5. Perlahan migrate/fix data lama
   ↓
6. VALIDATE CONSTRAINT untuk enforce ke semua data
```

Pendekatan ini memungkinkan sistem tetap berjalan tanpa downtime besar, sambil memastikan kualitas data meningkat secara bertahap.

#### Perbandingan dengan Check Constraint

Perbedaan perilaku antara domain dan check constraint menjadi sangat jelas dalam konteks migrasi.

```sql
-- Check Constraint: Tidak bisa gradual
ALTER TABLE products ADD CHECK (price > 0);
-- Langsung validasi SEMUA data existing
-- Jika ada data invalid → ERROR immediately!
```

Check constraint di level tabel akan langsung memeriksa seluruh data yang ada. Jika satu saja baris tidak valid, perintah akan gagal.

```sql
-- Domain: Bisa gradual
ALTER DOMAIN positive_amount
    ADD CHECK (VALUE > 0)
    NOT VALID;
-- Data lama: Tidak di-check (dengan caveat race condition)
-- Data baru: Divalidasi
```

Inilah salah satu alasan kuat mengapa domain sangat cocok untuk skema yang berkembang secara bertahap: ia memberi kita kontrol lebih halus atas **kapan** dan **bagaimana** aturan validasi ditegakkan.

### h. Automatic Type Downcast - Behavior Penting!

Pada bagian ini, kita membahas perilaku PostgreSQL yang **sering tidak disadari**, tetapi sangat penting ketika bekerja dengan domain. Perilaku ini dikenal sebagai **automatic type downcast**.

#### Apa itu Automatic Type Downcast?

Ketika sebuah **operator atau function** dari _underlying type_ (tipe dasar domain) diaplikasikan ke nilai bertipe domain, PostgreSQL akan **secara otomatis menurunkan tipe domain tersebut ke base type-nya**. Akibatnya, hasil operasi **bukan lagi domain**, melainkan tipe data dasarnya.

#### Contoh Dasar Automatic Downcast

```sql
CREATE DOMAIN posint AS INTEGER CHECK (VALUE > 0);

CREATE TABLE mytable (id posint);
INSERT INTO mytable VALUES (5);
```

Kolom `id` bertipe `posint`, yang berarti nilainya selalu harus lebih besar dari 0.

```sql
-- Query ini menghasilkan INTEGER, BUKAN posint!
SELECT id - 1 FROM mytable;  -- Hasil: INTEGER (bukan posint)
```

Meskipun `id` adalah `posint`, operasi aritmatika `id - 1` menggunakan operator milik `INTEGER`. PostgreSQL otomatis melakukan downcast ke `INTEGER`, sehingga hasilnya juga bertipe `INTEGER`.

#### Implikasi Penting terhadap Constraint Domain

Karena hasil operasi sudah bukan domain lagi, **constraint domain tidak ikut diperiksa ulang**.

```sql
SELECT (id - 1) FROM mytable WHERE id = 1;
-- Hasil: 0 (INTEGER)
-- constraint VALUE > 0 TIDAK di-check!
```

Secara logika, nilai `0` seharusnya tidak valid untuk `posint`. Namun karena hasil query adalah `INTEGER`, bukan `posint`, constraint domain tidak berlaku.

Jika kita ingin constraint domain diterapkan kembali, maka **harus melakukan cast secara eksplisit**.

```sql
SELECT (id - 1)::posint FROM mytable WHERE id = 1;
-- ERROR: value for domain posint violates check constraint
```

Dengan cast ke `posint`, PostgreSQL kembali menerapkan aturan `VALUE > 0`, dan hasil yang tidak valid langsung ditolak.

#### Dampak Automatic Downcast pada Domain Non-Numerik

Perilaku ini tidak hanya berlaku untuk angka, tetapi juga untuk domain berbasis teks.

```sql
CREATE DOMAIN email AS TEXT CHECK (VALUE LIKE '%@%');

CREATE TABLE users (
    user_email email
);
```

Kolom `user_email` dijamin memiliki format email selama nilainya tetap bertipe `email`.

```sql
-- Substring operation: Downcast ke TEXT
SELECT substring(user_email, 1, 5) FROM users;
-- Hasil: TEXT (bukan email)
-- Constraint email TIDAK berlaku lagi!
```

Fungsi `substring` adalah fungsi milik `TEXT`, sehingga hasilnya otomatis bertipe `TEXT`. Pada titik ini, aturan validasi email sudah tidak relevan lagi.

```sql
-- Ini bisa diassign ke kolom TEXT tanpa validasi
CREATE TEMP TABLE temp AS
    SELECT substring(user_email, 1, 5) AS partial FROM users;
-- partial adalah TEXT, bukan email
```

Kolom `partial` tidak memiliki jaminan format email sama sekali, karena domain sudah “hilang” akibat downcast.

#### Kenapa Perilaku Ini Sangat Penting?

Automatic downcast berarti:

- Domain **menjaga validitas data saat disimpan**
- Tetapi **tidak menjamin validitas hasil ekspresi atau operasi**
- Constraint domain **tidak bersifat transitive** melalui operasi atau function

Kesimpulannya, domain sangat efektif untuk memastikan **integritas data pada kolom**, tetapi ketika data tersebut diproses lebih lanjut dalam ekspresi SQL, kita harus sadar bahwa hasilnya bisa keluar dari perlindungan domain. Jika constraint domain perlu ditegakkan kembali, maka **explicit cast ke domain adalah satu-satunya cara**.

### i. Container Type Limitations - PENTING!

Pada bagian ini, kita membahas keterbatasan domain yang **sangat penting diketahui**, terutama ketika domain digunakan di dalam _container type_. Jika batasan ini tidak dipahami sejak awal, perubahan skema di kemudian hari bisa menjadi sangat sulit dan berisiko.

#### Jenis Container Type yang Bermasalah

Operasi berikut pada domain:

- `ALTER DOMAIN ADD CONSTRAINT`
- `VALIDATE CONSTRAINT`
- `SET NOT NULL`

akan **gagal** jika domain (termasuk _derived domain_) digunakan di dalam:

- Kolom bertipe **composite type** (row type)
- Kolom bertipe **array**
- Kolom bertipe **range**

Artinya, selama domain tersebut berada di dalam salah satu container type ini, PostgreSQL tidak mengizinkan perubahan struktur domain.

#### Contoh Kasus yang Akan Gagal

```sql
CREATE DOMAIN posint AS INTEGER CHECK (VALUE > 0);
```

Domain `posint` terlihat sederhana dan aman. Masalah baru muncul ketika domain ini digunakan di dalam composite type.

```sql
-- Domain dalam composite type
CREATE TYPE address_type AS (
    street TEXT,
    zipcode posint  -- Domain dalam composite
);

CREATE TABLE locations (
    location address_type
);
```

Pada titik ini, `posint` sudah “terkunci” di dalam struktur composite. Ketika kita mencoba memodifikasinya, PostgreSQL akan menolak.

```sql
-- INI AKAN GAGAL:
ALTER DOMAIN posint ADD CONSTRAINT max_val CHECK (VALUE < 1000);
-- ERROR: cannot alter type "posint" because column "locations.location" uses it
```

Masalah yang sama juga terjadi ketika domain digunakan sebagai elemen array.

```sql
-- Domain dalam array
CREATE TABLE numbers (
    values posint[]  -- Array of domain
);
```

Jika kita mencoba mengubah properti domain, hasilnya juga gagal.

```sql
-- INI JUGA AKAN GAGAL:
ALTER DOMAIN posint SET NOT NULL;
-- ERROR: cannot alter type "posint" because it is used in array type
```

PostgreSQL melarang perubahan ini karena domain sudah menjadi bagian dari struktur tipe yang lebih kompleks, dan mengubahnya berpotensi merusak integritas internal tipe tersebut.

#### Dampak Praktis dari Keterbatasan Ini

Keterbatasan ini berarti:

- Domain **masih aman dan fleksibel** saat digunakan langsung sebagai tipe kolom
- Tetapi menjadi **sangat sulit dimodifikasi** jika sudah dipakai di array, composite, atau range
- Perubahan domain di konteks ini hampir selalu memerlukan perubahan struktur tabel

Karena itu, penggunaan domain di dalam container type harus dipertimbangkan dengan sangat hati-hati.

#### Workaround yang Bisa Dilakukan

Jika terlanjur perlu mengubah domain yang sudah digunakan dalam container type, satu-satunya jalan adalah melakukan langkah yang cukup berat.

```sql
-- Jika perlu ALTER domain dalam container type:
-- 1. Backup data
-- 2. Drop tables/columns yang pakai container type
-- 3. ALTER domain
-- 4. Recreate tables/columns
-- 5. Restore data
```

Langkah ini jelas tidak ideal, terutama untuk sistem produksi dengan data besar.

Alternatif yang lebih aman adalah **tidak menggunakan domain di dalam container type**, dan kembali ke pendekatan base type dengan check constraint di level tabel.

Atau, gunakan base type + check constraint untuk container types. Dengan cara ini, validasi tetap ada, tetapi perubahan aturan di masa depan menjadi jauh lebih mudah dan tidak terhalang oleh keterbatasan container type.

### j. Keuntungan Domain

Domain bukan sekadar fitur tambahan, tetapi alat desain skema yang membawa banyak keuntungan praktis. Jika digunakan dengan tepat, domain membantu menjaga kualitas data, menyederhanakan perubahan skema, dan meningkatkan kejelasan komunikasi di dalam tim.

#### 1. Reusability dan Konsistensi

Salah satu manfaat terbesar domain adalah kemampuannya untuk **didefinisikan sekali dan digunakan di banyak tempat** dengan aturan yang selalu konsisten.

```sql
-- Define sekali
CREATE DOMAIN email AS TEXT
    CONSTRAINT valid_email
    CHECK (VALUE ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$');
```

Domain `email` di atas mendefinisikan format email yang valid menggunakan regular expression.

```sql
-- Pakai di banyak tempat
CREATE TABLE users (
    primary_email email,
    recovery_email email
);

CREATE TABLE contacts (
    business_email email,
    personal_email email
);

CREATE TABLE newsletter_subscribers (
    subscriber_email email
);
```

Semua kolom yang menggunakan domain `email` otomatis memiliki **aturan validasi yang sama**, tanpa perlu menuliskannya ulang. Dengan pendekatan ini, tidak ada lagi perbedaan aturan validasi antar tabel hanya karena kelalaian atau copy–paste yang tidak konsisten.

#### 2. Maintainability

Ketika aturan bisnis berubah, domain membuat proses perubahan menjadi jauh lebih sederhana.

```sql
-- Jika rule email berubah, cukup ALTER domain sekali
ALTER DOMAIN email DROP CONSTRAINT valid_email;
ALTER DOMAIN email ADD CONSTRAINT valid_email
    CHECK (VALUE ~ '^[new_regex]$');
```

Dengan mengubah domain satu kali, **seluruh tabel dan kolom yang menggunakannya langsung mengikuti aturan baru**. Tidak perlu melakukan `ALTER TABLE` di banyak tempat, yang biasanya rawan kesalahan dan memakan waktu.

#### 3. Self-Documenting Code

Domain juga membantu membuat skema database menjadi lebih **mudah dibaca dan dipahami**, bahkan tanpa dokumentasi tambahan.

```sql
-- ❌ Kurang jelas
CREATE TABLE orders (
    amount NUMERIC CHECK (amount > 0)  -- Amount apa? Positive kenapa?
);
```

Pada contoh ini, kita tahu `amount` harus positif, tetapi tidak jelas konteks bisnisnya.

```sql
-- ✅ Lebih jelas dan self-documenting
CREATE DOMAIN monetary_amount AS NUMERIC CHECK (VALUE > 0);

CREATE TABLE orders (
    amount monetary_amount  -- Jelas: ini monetary amount dengan validasi built-in
);
```

Nama domain `monetary_amount` langsung menjelaskan bahwa kolom tersebut merepresentasikan nilai uang, lengkap dengan validasi yang relevan. Skema menjadi **self-documenting**, sehingga lebih mudah dipahami oleh siapa pun yang membacanya.

#### 4. Team Communication

Dalam kerja tim, domain berfungsi sebagai **kontrak bersama** yang disepakati.

```sql
-- Domain sebagai kontrak dalam tim
CREATE DOMAIN product_sku AS TEXT
    CONSTRAINT sku_format
    CHECK (VALUE ~ '^[A-Z]{3}-[0-9]{5}$');
```

Domain ini secara eksplisit mendefinisikan format SKU produk.

```sql
-- Developer baru lihat schema:
CREATE TABLE products (
    sku product_sku  -- Langsung paham format SKU tanpa baca dokumentasi
);
```

Seorang developer baru yang melihat skema ini langsung memahami bahwa:

- `sku` memiliki format tertentu
- Format tersebut dijaga oleh database, bukan hanya oleh aplikasi

Dengan demikian, domain tidak hanya menjaga integritas data, tetapi juga meningkatkan **kejelasan komunikasi dan kolaborasi** dalam tim pengembang.

## 3. Hubungan Antar Konsep

### Evolusi Validasi Data:

```
Level 1: Tipe Data Dasar
    INTEGER, TEXT, NUMERIC
    ↓ (Tidak cukup spesifik)

Level 2: Check Constraints
    CHECK (price > 0)
    CHECK (length(code) = 5)
    ↓ (Duplikasi di banyak tempat)

Level 3: Domains
    CREATE DOMAIN positive_price AS NUMERIC CHECK (VALUE > 0)
    ↓ (Reusable, maintainable)

Level 4: Derived Domains
    CREATE DOMAIN limited_price AS positive_price CHECK (VALUE < 1000)
    ↓ (Layered validation)
```

### Domain dalam Ekosistem PostgreSQL:

```
Base Types (INT, TEXT, etc)
    ↓ wrapped by
Domains (custom types with validation)
    ↓ can be base for
Other Domains (derived domains)
    ↓ used in
Tables & Columns (but NOT container types for ALTER)
    ↓ downcast to base type when
Operations/Functions applied
    ↓ protected by
Check Constraints (for multi-column rules)
    ↓ ensures
Data Integrity
```

### Kapan Menggunakan Apa:

```
Single column, used once
    → Check Constraint inline

Single column, used multiple times, NOT in container types
    → Domain

Single column IN container types (composite, array, range)
    → Base type + Check Constraint (easier to maintain)

Multiple columns relationship
    → Table-level Check Constraint

Complex business logic that changes often
    → Application layer
```

### Kombinasi Domain + Check Constraint:

```sql
-- Strategi optimal: Gunakan keduanya
CREATE DOMAIN positive_amount AS NUMERIC CHECK (VALUE > 0);
CREATE DOMAIN email AS TEXT CHECK (VALUE LIKE '%@%');
CREATE DOMAIN us_postal_code AS TEXT
    CHECK (VALUE ~ '^\d{5}(-\d{4})?$');

CREATE TABLE orders (
    -- Domain untuk single-column validation
    customer_email email,
    billing_email email,
    price positive_amount,
    discount positive_amount,
    shipping us_postal_code,

    -- Check constraint untuk multi-column logic
    CHECK (billing_email IS NOT NULL OR customer_email IS NOT NULL),
    CHECK (price > discount),
    CHECK (discount <= price * 0.5)  -- Max 50% discount
);
```

## 4. Kesimpulan

Domain Types adalah fitur powerful PostgreSQL yang mengenkapsulasi tipe data dan check constraints menjadi custom type yang reusable. Domain mengatasi masalah duplikasi check constraint dan meningkatkan konsistensi validasi di seluruh database.

**Key Points:**

1. **Definisi**: Domain = Base Type + Check Constraint(s) + Custom Name
2. **Syntax**: Menggunakan keyword `VALUE` untuk mereferensi nilai yang divalidasi
3. **Reusability**: Satu domain bisa dipakai di banyak kolom dan tabel
4. **Maintainability**: Bisa di-ALTER, tidak perlu DROP-RECREATE
5. **Gradual Migration**: Fitur `NOT VALID` untuk rollout bertahap

**Kapan Menggunakan Domain:**

- ✅ Validasi yang sama dipakai di banyak tempat
- ✅ Format data yang standard (email, phone, postal code)
- ✅ Business rules yang konsisten (positive amount, valid percentage)
- ✅ Komunikasi yang jelas dalam tim (self-documenting)

**Kapan TIDAK Menggunakan Domain:**

- ❌ Multi-column validation (pakai check constraint)
- ❌ Logic yang sangat complex atau sering berubah
- ❌ Validasi yang hanya dipakai sekali

**Best Practice:**

- Kombinasikan domain (untuk single-column) dengan check constraint (untuk multi-column)
- Beri nama yang descriptive dan meaningful
- Gunakan `NOT VALID` untuk migration data existing
- Dokumentasikan domain di schema documentation

**Key takeaway:** Domain adalah abstraksi yang meningkatkan reusability, maintainability, dan konsistensi validasi data. Mereka adalah "DRY principle" untuk database constraints - Define once, use everywhere, maintain easily.

```

```
