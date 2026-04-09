# 📝 Character Sets dan Collations di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang character sets (encoding) dan collations di PostgreSQL, dua konsep penting yang menentukan bagaimana karakter disimpan dan dibandingkan. Character set mendefinisikan karakter apa saja yang legal/valid, sedangkan collation mendefinisikan aturan bagaimana karakter-karakter tersebut saling berhubungan (misalnya apakah 'a' sama dengan 'A'). Video juga menjelaskan cara membuat custom collation dan kapan sebaiknya menggunakan alternatif lain seperti functional indexes.

## 2. Konsep Utama

### a. Character Set / Encoding

Character set (sering juga disebut encoding atau charset) adalah aturan yang menentukan bagaimana karakter yang kita lihat di layar direpresentasikan dalam bentuk bytes saat disimpan di media penyimpanan seperti disk. Komputer sebenarnya tidak menyimpan huruf, angka, atau emoji secara langsung, melainkan menyimpannya sebagai angka biner (bytes). Encoding berperan sebagai “kamus penerjemah” antara karakter manusia dan representasi byte di sistem.

#### Konsep Dasar

Alur kerjanya dapat dipahami secara sederhana sebagai berikut:

```
Character on screen → Encoding → Bytes on disk
     'A'           →  UTF-8   →  0x41
     '😀'          →  UTF-8   →  0xF0 0x9F 0x98 0x80
```

Karakter `'A'` direpresentasikan dengan satu byte (`0x41`) karena masih berada dalam rentang ASCII. Sebaliknya, emoji seperti `'😀'` membutuhkan beberapa byte karena berada di luar rentang karakter dasar ASCII. Encoding menentukan bagaimana proses pemetaan ini dilakukan.

#### UTF-8: The Recommended Choice

UTF-8 adalah encoding yang paling direkomendasikan dan paling umum digunakan saat ini, termasuk di PostgreSQL. UTF-8 merupakan encoding multi-byte, artinya satu karakter dapat menggunakan 1 hingga 4 byte, tergantung jenis karakternya. Karakter sederhana menggunakan 1 byte, sedangkan karakter kompleks (misalnya emoji atau huruf dari bahasa tertentu) menggunakan lebih banyak byte.

Untuk melihat encoding yang digunakan oleh database PostgreSQL, kita bisa menjalankan perintah berikut:

```sql
-- Melihat encoding database
\l  -- Atau SELECT datname, encoding FROM pg_database;
```

Contoh output-nya kira-kira seperti ini:

```text
-- Name    | Encoding | Collate      | Ctype
-- mydb    | UTF8     | en_US.UTF-8  | en_US.UTF-8
```

Dari sini terlihat bahwa database `mydb` menggunakan encoding `UTF8`, yang berarti seluruh data teks di database tersebut akan disimpan dan diproses menggunakan aturan UTF-8.

#### Karakteristik UTF-8

UTF-8 dipilih secara luas karena memiliki banyak keunggulan penting:

- Mendukung seluruh rentang Unicode, sehingga hampir semua bahasa di dunia dapat direpresentasikan.
- Mendukung karakter beraksen seperti `é`, `ñ`, dan `ü`.
- Membedakan huruf besar dan kecil (uppercase dan lowercase).
- Mendukung simbol modern seperti emoji (`😀`, `🎉`, `🔥`).
- Menggunakan lebar byte variabel (1–4 byte per karakter), sehingga efisien untuk teks ASCII sekaligus fleksibel untuk karakter kompleks.
- Tetap kompatibel ke belakang dengan ASCII, sehingga teks lama tetap bisa dibaca tanpa masalah.

#### Catatan Naming

Di PostgreSQL, penamaan encoding cukup fleksibel. `UTF8` adalah nama canonical, sedangkan `UTF-8` hanyalah alias. Keduanya diperlakukan sebagai encoding yang sama, jadi tidak ada perbedaan fungsional di antara keduanya.

#### Contoh Penggunaan

Berikut contoh tabel dan data yang menunjukkan fleksibilitas UTF-8:

```sql
-- UTF-8 bisa handle semua ini:
CREATE TABLE messages (
    content TEXT
);

INSERT INTO messages VALUES ('Hello World');           -- ASCII
INSERT INTO messages VALUES ('Café résumé');          -- Accented
INSERT INTO messages VALUES ('こんにちは');             -- Japanese
INSERT INTO messages VALUES ('مرحبا');                 -- Arabic
INSERT INTO messages VALUES ('Hello 😀🎉');           -- Emoji
-- Semua berhasil dengan UTF-8!
```

Semua `INSERT` di atas berhasil karena UTF-8 mampu menangani berbagai jenis karakter dari berbagai bahasa dan simbol.

Sebagai perbandingan, jika menggunakan encoding yang lebih terbatas seperti `LATIN1`, akan muncul masalah ketika mencoba menyimpan karakter di luar kemampuannya:

```sql
-- Dengan encoding lain (misalnya Latin1):
SET client_encoding = 'LATIN1';
INSERT INTO messages VALUES ('こんにちは');  -- ❌ ERROR!
-- ERROR: character with byte sequence 0xe3 0x81 0x93 in encoding "UTF8"
--        has no equivalent in encoding "LATIN1"
```

Error tersebut muncul karena karakter Jepang tidak memiliki representasi yang valid dalam encoding `LATIN1`. Inilah alasan utama mengapa UTF-8 hampir selalu menjadi pilihan terbaik: ia meminimalkan risiko error encoding dan memastikan data teks tetap konsisten, fleksibel, dan future-proof.

### b. Collation (Kolasi)

Collation adalah sekumpulan aturan yang menentukan bagaimana karakter dibandingkan satu sama lain dan bagaimana urutan mereka ditentukan. Jika encoding berfokus pada bagaimana karakter disimpan sebagai byte, maka collation berfokus pada bagaimana karakter tersebut diperlakukan saat dilakukan perbandingan, pencarian, dan pengurutan data teks.

Dengan kata lain, collation memengaruhi logika “bahasa” yang digunakan database ketika bekerja dengan data bertipe teks.

#### Apa yang Ditentukan oleh Collation

Collation mengatur beberapa aspek penting dalam pemrosesan string, antara lain:

- Sensitivitas huruf besar dan kecil (case sensitivity), misalnya apakah `'a'` dianggap sama dengan `'A'`.
- Sensitivitas terhadap aksen (accent sensitivity), misalnya apakah `'e'` dianggap sama dengan `'é'`.
- Aturan pengurutan karakter, seperti apakah `'a'` diurutkan sebelum `'b'`, dan seterusnya.
- Cara karakter spesial atau non-alfabet (misalnya simbol atau karakter Unicode tertentu) diurutkan.

Semua aturan ini sangat berpengaruh pada operasi seperti `ORDER BY`, `GROUP BY`, perbandingan menggunakan operator `=`, serta pencarian teks.

#### Melihat Collation Default Database

Setiap database PostgreSQL memiliki collation default yang biasanya ditentukan saat database dibuat, dan sering kali mengikuti locale sistem operasi. Kita dapat melihat collation default dengan dua cara berikut.

Dari command line PostgreSQL:

```sql
\l
```

Atau menggunakan query SQL:

```sql
SELECT datname, datcollate, datctype FROM pg_database;
```

Output yang umum dijumpai misalnya:

```text
en_US.UTF-8
```

Penjelasannya:

- `en_US` menunjukkan locale bahasa Inggris dengan wilayah Amerika Serikat.
- `UTF-8` menunjukkan encoding karakter yang digunakan.

Locale ini memengaruhi cara database membandingkan dan mengurutkan string sesuai aturan bahasa dan budaya yang berlaku.

#### Karakteristik Collation en_US.UTF-8

Collation `en_US.UTF-8` memiliki beberapa perilaku khas yang penting untuk dipahami.

##### Case Sensitive

Pada collation ini, huruf besar dan huruf kecil dianggap berbeda. Contohnya:

```sql
SELECT 'abc' = 'ABC';  -- FALSE
```

Hasilnya `FALSE` karena `'abc'` (huruf kecil) tidak dianggap sama dengan `'ABC'` (huruf besar).

Sebaliknya, jika kedua string identik secara penulisan:

```sql
SELECT 'abc' = 'abc';  -- TRUE
```

Maka hasilnya `TRUE`, karena kedua nilai benar-benar sama.

Contoh lain:

```sql
SELECT 'Hello' = 'hello';  -- FALSE
```

Ini menegaskan bahwa perbedaan huruf besar dan kecil berpengaruh langsung pada hasil perbandingan.

##### Accent Sensitive

Selain case sensitivity, collation `en_US.UTF-8` juga sensitif terhadap aksen. Artinya, karakter tanpa aksen dianggap berbeda dari karakter dengan aksen.

```sql
SELECT 'cafe' = 'café';  -- FALSE
```

Hasil `FALSE` muncul karena `'e'` dan `'é'` adalah dua karakter yang berbeda menurut aturan collation ini.

Pemahaman tentang perilaku collation sangat penting, terutama ketika mendesain sistem yang melibatkan pencarian teks, pengurutan data, atau perbandingan string lintas bahasa. Kesalahan memilih collation dapat menghasilkan hasil query yang tidak sesuai ekspektasi, meskipun datanya terlihat “mirip” secara visual.

### c. Server Encoding vs Client Encoding

Dalam PostgreSQL, proses penyimpanan dan pertukaran data teks melibatkan dua jenis encoding yang berbeda namun saling berkaitan, yaitu server encoding dan client encoding. Memahami perbedaan dan hubungan keduanya sangat penting untuk mencegah error, karakter rusak, atau bahkan kehilangan data.

#### 1. Server Encoding

Server encoding adalah encoding yang digunakan oleh database untuk menyimpan data di disk. Semua data teks di dalam database harus mengikuti aturan encoding ini. Server encoding ditentukan saat database dibuat dan tidak bisa diubah setelahnya.

Untuk melihat server encoding yang sedang digunakan, kita dapat menjalankan perintah berikut:

```sql
SHOW server_encoding;
-- Output: UTF8
```

Contoh pembuatan database dengan server encoding UTF-8:

```sql
CREATE DATABASE mydb
    ENCODING 'UTF8'
    LC_COLLATE 'en_US.UTF-8'
    LC_CTYPE 'en_US.UTF-8';
```

Pada contoh di atas, database `mydb` diset untuk menggunakan UTF-8 sebagai encoding penyimpanan. Artinya, semua karakter—mulai dari ASCII, huruf beraksen, hingga emoji—akan disimpan dalam format UTF-8 di disk.

#### 2. Client Encoding

Client encoding adalah encoding yang digunakan oleh client, seperti aplikasi, driver, atau tool (misalnya `psql`), saat berkomunikasi dengan PostgreSQL server. Client encoding menentukan bagaimana data teks dikirim ke server dan bagaimana hasil query dikirim kembali ke client.

Untuk melihat client encoding yang sedang aktif:

```sql
SHOW client_encoding;
-- Output: UTF8
```

Client encoding dapat diubah selama sesi koneksi berjalan, misalnya:

```sql
SET client_encoding = 'LATIN1';
```

Perubahan ini hanya memengaruhi cara data dikirim dan diterima oleh client, bukan cara data disimpan di database.

#### Proses Konversi Encoding

Ketika client encoding berbeda dengan server encoding, PostgreSQL akan mencoba melakukan konversi secara otomatis. Alurnya kira-kira seperti berikut:

```
Client (LATIN1) → PostgreSQL Server → Conversion → Database (UTF8)
     ↓                                                      ↓
  "café"           Attempt to convert               Store as UTF8
```

Dalam contoh ini, teks `"café"` dikirim oleh client dengan encoding `LATIN1`. PostgreSQL kemudian mencoba mengonversinya ke UTF-8 sebelum menyimpannya ke database. Jika konversi berhasil, data akan disimpan dengan benar.

#### Potensi Masalah

Masalah muncul ketika client encoding dan server encoding tidak kompatibel, atau ketika karakter yang dikirim client tidak dapat direpresentasikan dalam encoding tujuan.

Contoh skenario bermasalah:

```sql
SET client_encoding = 'SQL_ASCII';  -- Very limited encoding

-- Mencoba insert emoji
INSERT INTO messages VALUES ('Hello 😀');
```

Dalam kasus ini, PostgreSQL akan menghasilkan error atau berpotensi menyebabkan data loss, karena encoding `SQL_ASCII` tidak mampu merepresentasikan emoji. PostgreSQL tidak dapat mengonversi karakter tersebut ke server encoding dengan aman.

Best practice yang sangat disarankan adalah memastikan client encoding dan server encoding sama-sama menggunakan UTF-8:

```sql
SET client_encoding = 'UTF8';  -- Match server encoding
```

Dengan cara ini, PostgreSQL tidak perlu melakukan konversi yang berisiko, dan seluruh karakter Unicode dapat ditangani dengan konsisten.

#### Masalah Overlap Encoding

Untuk memahami kenapa konversi bisa gagal, bayangkan dua encoding dengan himpunan karakter yang berbeda:

```
Encoding A: {a, b, c, é, ñ}
Encoding B: {a, b, c, x, y, z}
```

Karakter yang sama-sama dimiliki oleh kedua encoding (`a`, `b`, `c`) dapat dikonversi tanpa masalah. Namun karakter yang hanya ada di Encoding A (`é`, `ñ`) tidak memiliki padanan di Encoding B.

```
Overlap: {a, b, c}  ✅ Bisa convert
Not in B: {é, ñ}    ❌ Tidak bisa convert → Error atau data loss
```

Inilah inti dari banyak masalah encoding: selama karakter berada di area overlap, semuanya tampak baik-baik saja. Begitu karakter di luar overlap digunakan, barulah error muncul. Karena itu, menggunakan UTF-8 secara konsisten di sisi server dan client adalah pendekatan paling aman dan paling fleksibel untuk sistem modern.

### d. Case Sensitivity dengan Collation

Pada PostgreSQL, perilaku perbandingan huruf besar dan kecil sangat dipengaruhi oleh collation yang digunakan. Secara default, collation `en_US.UTF-8` bersifat case-sensitive, artinya huruf besar dan huruf kecil dianggap sebagai karakter yang berbeda. Hal ini sering kali terlihat sepele, tetapi dampaknya sangat nyata dalam query sehari-hari.

#### Perbandingan String dengan Explicit Collation

Kita bisa secara eksplisit menyebutkan collation saat melakukan perbandingan string untuk memperjelas perilakunya.

```sql
SELECT 'abc' = 'ABC' COLLATE "en_US.UTF-8";  -- FALSE
```

Hasil `FALSE` muncul karena meskipun hurufnya sama, perbedaan case (`abc` vs `ABC`) membuat PostgreSQL menganggap keduanya tidak sama.

```sql
SELECT 'abc' = 'abc' COLLATE "en_US.UTF-8";  -- TRUE
```

Pada contoh ini, kedua string identik, baik dari segi huruf maupun case, sehingga hasilnya `TRUE`.

```sql
SELECT 'abc' = 'Abc' COLLATE "en_US.UTF-8";  -- FALSE
```

Bahkan hanya satu huruf kapital di awal sudah cukup untuk membuat perbandingan bernilai `FALSE`. Ini menegaskan bahwa pada collation default, PostgreSQL benar-benar membedakan huruf besar dan kecil.

#### Dampak di Dunia Nyata

Perilaku case-sensitive ini sangat sering memengaruhi aplikasi nyata, terutama pada data seperti username atau email.

Perhatikan contoh tabel berikut:

```sql
CREATE TABLE users (
    email TEXT
);

INSERT INTO users VALUES ('john@example.com');
```

Data yang disimpan menggunakan huruf kecil. Namun, saat melakukan pencarian dengan case yang berbeda:

```sql
SELECT * FROM users WHERE email = 'John@example.com';
```

Query tersebut tidak mengembalikan hasil apa pun. Ini bukan karena datanya tidak ada, melainkan karena perbedaan huruf besar dan kecil membuat nilai `'John@example.com'` tidak dianggap sama dengan `'john@example.com'`.

#### Solusi Umum

Ada beberapa pendekatan yang umum digunakan untuk menangani masalah ini, tergantung kebutuhan aplikasi.

##### 1. Menyamakan Case Saat Query

Pendekatan paling sederhana adalah mengubah kedua sisi perbandingan menjadi format yang sama, biasanya huruf kecil.

```sql
SELECT * FROM users
WHERE LOWER(email) = LOWER('John@example.com');
```

Dengan cara ini, perbandingan menjadi konsisten karena baik data di kolom maupun input query diproses dalam bentuk lowercase.

##### 2. Menggunakan ILIKE

PostgreSQL menyediakan operator `ILIKE`, yang melakukan pencocokan string tanpa memperhatikan case.

```sql
SELECT * FROM users
WHERE email ILIKE 'John@example.com';
```

Operator ini sangat praktis untuk pencarian case-insensitive, terutama ketika digunakan dalam query pencarian teks.

##### 3. Menggunakan Custom Collation

Alternatif yang lebih struktural adalah menggunakan collation khusus yang bersifat case-insensitive. Pendekatan ini biasanya diterapkan pada level kolom atau database, sehingga perbandingan case-insensitive menjadi perilaku default. Topik ini akan dibahas lebih lanjut pada bagian berikutnya.

### e. Membuat Custom Collation

PostgreSQL menyediakan fleksibilitas tinggi dalam hal collation, termasuk kemampuan untuk membuat collation dengan aturan khusus sesuai kebutuhan aplikasi. Custom collation sangat berguna ketika perilaku default—misalnya case-sensitive—tidak sesuai dengan kebutuhan bisnis, seperti pencarian email atau username yang seharusnya tidak membedakan huruf besar dan kecil.

#### Collation Providers

Saat membuat custom collation, PostgreSQL mendukung dua jenis provider utama:

- **ICU (International Components for Unicode)**
  Provider ini sangat direkomendasikan untuk custom collation. ICU bersifat lebih konsisten dan portable, sehingga perilakunya cenderung sama di berbagai sistem operasi. ICU juga menyediakan kontrol yang sangat detail terhadap aturan perbandingan karakter.

- **libc**
  Provider ini bergantung pada library sistem operasi. Karena itu, perilaku collation bisa berbeda antara Linux, macOS, atau sistem lainnya. Untuk aplikasi lintas platform, opsi ini umumnya kurang disarankan.

#### Sintaks Dasar Membuat Collation

Struktur dasar pembuatan custom collation di PostgreSQL adalah sebagai berikut:

```sql
CREATE COLLATION collation_name (
    provider = icu,
    locale = 'locale_string',
    deterministic = false
);
```

Di sini, kita menentukan provider yang digunakan, locale yang mendefinisikan aturan perbandingan, serta apakah perbandingan bersifat deterministik atau tidak.

#### Contoh: Collation Case-Insensitive

Berikut contoh pembuatan collation yang bersifat case-insensitive menggunakan ICU:

```sql
CREATE COLLATION case_insensitive (
    provider = icu,
    locale = 'en-US-u-ks-level1',
    deterministic = false
);
```

Collation ini akan mengabaikan perbedaan huruf besar dan kecil, bahkan juga mengabaikan aksen.

Sebagai perbandingan, berikut alternatif menggunakan provider `libc`:

```sql
CREATE COLLATION case_insensitive_libc (
    provider = libc,
    locale = 'en_US.utf8'
);
```

Pendekatan ini bekerja, tetapi kurang portable karena hasilnya bisa berbeda tergantung sistem operasi yang digunakan.

#### Penjelasan Parameter ICU

Jika kita lihat lebih detail, parameter-parameter pada collation ICU memiliki makna sebagai berikut:

- `provider = icu` menunjukkan bahwa aturan perbandingan mengikuti standar International Components for Unicode.
- `locale = 'en-US'` menentukan bahasa Inggris dengan regional Amerika Serikat.
- `u-ks-level1` adalah Unicode extension yang mengatur tingkat sensitivitas perbandingan.
- `deterministic = false` memungkinkan perbandingan non-deterministik, yang dibutuhkan untuk collation yang mengabaikan perbedaan tertentu seperti case atau aksen.

Locale string `en-US-u-ks-level1` dapat dipecah menjadi bagian-bagian berikut:

```
en-US-u-ks-level1
│  │  │ │  │
│  │  │ │  └─ level1: Perbandingan karakter dasar saja
│  │  │ └──── ks: Key strength (tingkat sensitivitas)
│  │  └─────── u: Unicode extension
│  └────────── US: United States
└───────────── en: English
```

#### Levels of Sensitivity

Tingkat sensitivitas (level) menentukan seberapa detail perbandingan karakter dilakukan:

- **Level 1**
  Hanya membandingkan karakter dasar. Case dan aksen diabaikan.
  Contoh: `'a' = 'A' = 'á' = 'Á'`.

- **Level 2**
  Aksen diperhitungkan, tetapi case masih diabaikan.
  Contoh: `'a' = 'A'`, tetapi `'a' ≠ 'á'`.

- **Level 3**
  Case dan aksen diperhitungkan. Ini adalah perilaku default PostgreSQL.
  Contoh: `'a' ≠ 'A'`.

#### Menguji Custom Collation

Perbedaan perilaku sebelum dan sesudah menggunakan custom collation dapat dilihat dari contoh berikut:

```sql
-- Sebelum: Case sensitive
SELECT 'abc' = 'ABC';  -- FALSE
```

Dengan collation default, hasilnya `FALSE` karena perbedaan huruf besar dan kecil.

```sql
-- Setelah: Dengan custom collation
SELECT 'abc' = 'ABC' COLLATE case_insensitive;  -- TRUE
```

Sekarang hasilnya `TRUE` karena perbandingan dilakukan secara case-insensitive.

Karena menggunakan level 1, aksen juga diabaikan:

```sql
SELECT 'cafe' = 'café' COLLATE case_insensitive;  -- TRUE
```

#### Menggunakan Custom Collation pada Kolom

Custom collation akan jauh lebih praktis jika diterapkan langsung pada kolom, sehingga semua query otomatis mengikuti aturan tersebut.

```sql
CREATE TABLE users (
    email TEXT COLLATE case_insensitive,
    name_en TEXT COLLATE "en_US.UTF-8",      -- English names (case-sensitive)
    name_fr TEXT COLLATE "fr_FR.UTF-8",      -- French names
    name_de TEXT COLLATE "de_DE.UTF-8",      -- German names
);
```

Pada tabel ini, hanya kolom `email` yang menggunakan collation case-insensitive, sementara kolom `name` tetap menggunakan collation default.

```sql
INSERT INTO users VALUES ('John@Example.com', 'John Doe');
```

Sekarang, pencarian email tidak lagi bergantung pada case:

```sql
SELECT * FROM users WHERE email = 'john@example.com';  -- Ketemu
SELECT * FROM users WHERE email = 'JOHN@EXAMPLE.COM';  -- Ketemu juga
```

Pendekatan ini membuat query lebih sederhana, konsisten, dan aman, terutama untuk data yang secara logis seharusnya tidak membedakan huruf besar dan kecil, seperti email atau identifier pengguna.

### f. Alternatif untuk Case-Insensitive Search

Untuk kebutuhan pencarian data yang tidak membedakan huruf besar dan kecil (case-insensitive), membuat custom collation bukan satu-satunya solusi. Dalam praktik nyata, sering kali ada pendekatan yang lebih sederhana, lebih efisien, dan lebih mudah dioptimalkan dengan index. Berikut beberapa alternatif yang umum digunakan, beserta kelebihan dan konteks penggunaannya.

#### 1. Menyamakan Case Saat Penyimpanan (Downcase pada Input)

Pendekatan paling dasar adalah memastikan data disimpan dalam format yang konsisten, biasanya seluruhnya dalam huruf kecil. Dengan begitu, pencarian cukup membandingkan nilai yang sudah seragam.

Contoh menggunakan constraint untuk memaksa email selalu dalam lowercase:

```sql
CREATE TABLE users (
    email TEXT CHECK (email = LOWER(email))
);
```

Dengan constraint ini, PostgreSQL akan menolak data yang tidak sesuai, sehingga integritas data tetap terjaga.

Alternatif yang lebih fleksibel adalah menggunakan trigger untuk otomatis mengubah input menjadi lowercase sebelum disimpan:

```sql
CREATE OR REPLACE FUNCTION lowercase_email()
RETURNS TRIGGER AS $$
BEGIN
    NEW.email = LOWER(NEW.email);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER lowercase_email_trigger
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION lowercase_email();
```

Dengan pendekatan ini, aplikasi tidak perlu khawatir soal case, karena database akan memastikan nilai `email` selalu disimpan dalam bentuk lowercase.

#### 2. Generated Column (PostgreSQL 12+)

Mulai PostgreSQL 12, tersedia fitur generated column yang sangat berguna untuk pencarian case-insensitive yang efisien.

```sql
CREATE TABLE users (
    email TEXT,
    email_lower TEXT GENERATED ALWAYS AS (LOWER(email)) STORED
);
```

Kolom `email_lower` secara otomatis menyimpan versi lowercase dari `email`. Karena bersifat `STORED`, nilainya benar-benar disimpan di disk dan bisa diindeks.

```sql
CREATE INDEX idx_email_lower ON users(email_lower);
```

Dengan index ini, query pencarian akan sangat efisien:

```sql
SELECT * FROM users WHERE email_lower = LOWER('John@Example.com');
```

Pada query tersebut, PostgreSQL dapat menggunakan index secara langsung, sehingga performanya jauh lebih baik pada tabel berukuran besar.

#### 3. Functional Index

Jika tidak ingin menambah kolom baru, functional index adalah solusi yang praktis. Index ini dibuat berdasarkan hasil sebuah fungsi, bukan nilai kolom mentahnya.

```sql
CREATE INDEX idx_email_lower ON users (LOWER(email));
```

Agar index ini bisa digunakan, query harus menggunakan ekspresi yang sama:

```sql
SELECT * FROM users WHERE LOWER(email) = LOWER('John@Example.com');
```

Jika ekspresinya cocok, PostgreSQL akan memanfaatkan index tersebut. Pendekatan ini relatif ringan dan sering digunakan untuk kebutuhan exact match yang case-insensitive.

#### 4. ILIKE Operator

PostgreSQL menyediakan operator `ILIKE`, yaitu versi case-insensitive dari `LIKE`.

```sql
SELECT * FROM users WHERE email ILIKE 'john@example.com';
```

Secara fungsional, ini sangat mudah digunakan. Namun, perlu dicatat bahwa `ILIKE` tidak bisa memanfaatkan index B-tree biasa.

Untuk mengatasi keterbatasan ini, kita dapat menggunakan trigram index dengan ekstensi `pg_trgm`.

```sql
CREATE EXTENSION pg_trgm;
```

Setelah ekstensi aktif, buat index trigram:

```sql
CREATE INDEX idx_email_trgm ON users USING gin(email gin_trgm_ops);
```

Dengan index ini, query `ILIKE`—terutama yang menggunakan wildcard—bisa menjadi index-assisted:

```sql
SELECT * FROM users WHERE email ILIKE '%john%';
```

Untuk pencarian exact match yang sederhana, pendekatan functional index dengan `LOWER(email)` sering kali lebih efisien dan lebih mudah dipahami dibandingkan trigram index.

Secara umum, pilihan pendekatan tergantung pada kebutuhan: apakah fokus pada konsistensi data, kemudahan query, atau performa pencarian. Dalam banyak kasus, menyamakan case saat penyimpanan atau menggunakan functional index sudah cukup dan lebih sederhana dibandingkan membuat custom collation.

## 3. Hubungan Antar Konsep

### Hirarki Character Handling:

```
Database Level
    ├─ Default Encoding (UTF8)
    └─ Default Collation (en_US.UTF-8)
         ↓
Table Level
    ├─ Inherit atau override database defaults
    └─ Each column dapat specify sendiri
         ↓
Column Level
    ├─ Can have own encoding (sangat jarang, tidak recommended)
    └─ Can have own collation (lebih umum dan berguna)
         ↓
Query Level
    └─ COLLATE keyword untuk override sementara
```

### Encoding vs Collation:

```
Character Set (Encoding)
    "Apa karakter yang LEGAL?"
    ├─ UTF8: Semua Unicode ✅
    ├─ Latin1: Western European only
    └─ ASCII: Basic English only
         ↓
         Defines valid characters
         ↓
Collation
    "Bagaimana karakter RELATE?"
    ├─ Case sensitivity: 'a' vs 'A'
    ├─ Accent sensitivity: 'e' vs 'é'
    └─ Sorting order: 'a' < 'b' < 'c'
```

### Decision Tree untuk Case-Insensitive:

```
Butuh case-insensitive comparison?
├─ Untuk exact match (=)
│   ├─ Option 1: LOWER() pada input ✅ Simplest
│   ├─ Option 2: Generated column + index ✅ Best performance (PG 12+)
│   ├─ Option 3: Functional index ✅ Good balance
│   └─ Option 4: Custom collation ⚠️ Paling complex, use jika benar-benar perlu
└─ Untuk pattern match (LIKE)
    ├─ Option 1: ILIKE dengan trigram index ✅ For wildcard searches
    └─ Option 2: LOWER() + LIKE + functional index ✅ More control
```

### Layer Communication:

```
Application Layer
    ↓ (sends query)
Client Encoding (e.g., UTF8)
    ↓ (transmit over network)
PostgreSQL Server
    ↓ (conversion if needed)
Server Encoding (e.g., UTF8)
    ↓ (apply collation rules)
Collation (e.g., en_US.UTF-8)
    ↓ (store/retrieve)
Disk Storage (bytes)
```

## 5. Kesimpulan

Character sets dan collations adalah aspek fundamental dari bagaimana PostgreSQL menyimpan dan membandingkan text data. Dalam kebanyakan kasus, default settings (UTF8 encoding dengan en_US.UTF-8 collation) sudah sangat baik dan tidak perlu diubah.

**Key Points:**

1. **Encoding (Character Set)**: Mendefinisikan karakter apa saja yang legal (UTF8 recommended untuk semua kasus)
2. **Collation**: Mendefinisikan bagaimana karakter dibandingkan dan diurutkan (default case-sensitive)
3. **Server vs Client**: Ada dua encoding yang berbeda, PostgreSQL akan convert jika perlu dan overlap memungkinkan
4. **Custom Collation**: Bisa dibuat dengan ICU provider untuk case/accent-insensitive, tapi pertimbangkan performance impact
5. **Alternatives**: Untuk case-insensitive search, functional index, generated column, atau normalize input sering lebih baik dan lebih performant
6. **Provider Choice**: ICU provider lebih portable dan konsisten across platforms dibanding libc

**Best Practices:**

- ✅ Gunakan UTF8 encoding (support semua Unicode characters)
- ✅ Stick dengan default collation kecuali ada alasan kuat (en_US.UTF-8)
- ✅ Match client dan server encoding (keduanya UTF8) untuk menghindari conversion issues
- ✅ Untuk case-insensitive: Prefer functional index atau generated column (PostgreSQL 12+)
- ✅ Custom collation hanya untuk kasus spesifik yang tidak bisa diselesaikan cara lain
- ✅ Gunakan ICU provider untuk custom collations (lebih portable)
- ✅ Untuk ILIKE pattern matching, gunakan trigram index (pg_trgm extension)
- ✅ Test performance impact sebelum implement custom collation di production
- ✅ Document collation choices untuk team members

**Kapan Mengubah Defaults:**

- **Multi-language app** → Different collation per language column untuk correct sorting
- **Case-insensitive comparison** → Evaluate: functional index vs generated column vs custom collation
- **Legacy system integration** → Might need specific encoding (tapi migrate ke UTF8 jika memungkinkan)
- **Specific locale requirements** → Custom collation dengan ICU untuk aturan sorting spesifik

**Performance Hierarchy (fastest to slowest untuk case-insensitive search):**

1. **Normalized input + regular index** (e.g., store lowercase) - Fastest
2. **Generated column + index** (PostgreSQL 12+) - Very fast
3. **Functional index** (e.g., `CREATE INDEX ON t(LOWER(col))`) - Fast
4. **Custom collation** - Slower, tapi preserves original case
5. **ILIKE without index** - Slowest (sequential scan)

**Common Pitfalls to Avoid:**

- ❌ Assuming default collation is case-insensitive
- ❌ Using non-UTF8 encoding untuk new projects
- ❌ Creating custom collation when simpler solutions exist
- ❌ Forgetting `deterministic = false` untuk level1 collations
- ❌ Using LIKE instead of ILIKE untuk case-insensitive pattern matching
- ❌ Not creating appropriate indexes untuk case-insensitive searches
- ❌ Mixing client and server encodings without understanding overlap
- ❌ Applying custom collation ke semua columns (overhead tidak perlu)

**Decision Flow Chart:**

```
Perlu case-insensitive handling?
├─ YES
│   ├─ Need to preserve original case?
│   │   ├─ YES
│   │   │   ├─ PostgreSQL 12+? → Use generated column ✅ BEST
│   │   │   └─ PostgreSQL < 12? → Use functional index or custom collation
│   │   └─ NO → Normalize input (LOWER/UPPER) + constraint ✅ SIMPLEST
│   └─ Pattern matching (LIKE)?
│       ├─ Install pg_trgm extension
│       └─ Create trigram index ✅
└─ NO → Use default collation ✅ DEFAULT

Multi-language support?
├─ YES → Use per-column collations with ICU provider ✅
└─ NO → Use default collation ✅

Performance critical?
├─ YES → Benchmark all options, prefer generated column or functional index ✅
└─ NO → Custom collation acceptable ✅
```

**Key Takeaway:**

UTF8 encoding dengan en_US.UTF-8 collation adalah pilihan yang aman dan powerful untuk mayoritas aplikasi modern. Custom collation adalah tool yang powerful untuk kasus spesifik (terutama multi-language apps), tapi untuk use case umum seperti case-insensitive search, pertimbangkan alternatif seperti functional indexes atau generated columns yang lebih performant dan maintainable.

Pahami perbedaan fundamental antara encoding (apa yang legal) dan collation (bagaimana karakter relate) untuk menghindari "mystery bugs" dengan karakter special. Selalu test dengan real-world data yang include accented characters, emoji, dan multi-byte characters untuk ensure aplikasi handle semua edge cases dengan benar.

**Final Recommendations by Use Case:**

| Use Case                  | Recommendation                                | Why                           |
| ------------------------- | --------------------------------------------- | ----------------------------- |
| New project               | UTF8 + default collation                      | Covers 99% of needs           |
| Email addresses           | Normalize to lowercase + check constraint     | Simplest, fastest             |
| Usernames (preserve case) | Generated column (PG 12+) or functional index | Fast, maintains original      |
| Product SKUs              | Normalize to uppercase + unique constraint    | Simplest, consistent          |
| Multi-language content    | Per-column ICU collations                     | Correct sorting per language  |
| Search functionality      | pg_trgm + GIN index                           | Handles wildcards efficiently |
| Legacy system             | Match legacy encoding, plan UTF8 migration    | Compatibility, future-proof   |

**Migration Path:**

Jika saat ini menggunakan encoding lain atau tidak optimal setup:

```sql
-- 1. Assess current setup
SELECT datname, encoding, datcollate, datctype
FROM pg_database
WHERE datname = current_database();

-- 2. Create new database dengan UTF8
CREATE DATABASE myapp_new
    ENCODING 'UTF8'
    LC_COLLATE 'en_US.UTF-8'
    LC_CTYPE 'en_US.UTF-8';

-- 3. Dump dan restore data
-- pg_dump myapp_old | psql myapp_new

-- 4. Test thoroughly
-- 5. Switch over
```

Dengan memahami character sets dan collations dengan baik, developer dapat membuat aplikasi yang robust, internationalization-ready, dan performant dalam handling text data across berbagai bahasa dan character sets.
