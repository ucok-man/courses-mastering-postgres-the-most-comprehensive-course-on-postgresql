# 📝 Binary Data di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang penyimpanan binary data (data mentah dalam bentuk bytes) di PostgreSQL menggunakan tipe data `BYTEA`. Berbeda dengan TEXT yang menyimpan string dengan encoding dan collation, BYTEA menyimpan raw bytes tanpa interpretasi karakter. Video menjelaskan kapan sebaiknya menggunakan binary data, use cases yang valid (seperti checksums dan hashes), dan perbandingan ukuran storage antara berbagai representasi hash (MD5 sebagai text, BYTEA, dan UUID).

## 2. Konsep Utama

### a. Tipe Data BYTEA

BYTEA (byte array) adalah tipe data di PostgreSQL yang digunakan untuk menyimpan **binary data mentah**. Berbeda dengan tipe teks seperti `TEXT` atau `VARCHAR`, BYTEA tidak memperlakukan isinya sebagai karakter atau string, melainkan sebagai deretan byte apa adanya. Karena itu, BYTEA sangat cocok untuk menyimpan data seperti file biner, hasil enkripsi, gambar, atau format data non-teks lainnya.

#### Karakteristik BYTEA

BYTEA memiliki beberapa karakteristik penting yang perlu dipahami agar penggunaannya tepat:

- **Variable-length column**
  Ukuran kolom BYTEA bersifat dinamis, artinya panjang data menyesuaikan dengan jumlah byte yang disimpan.

- **Ukuran maksimum besar**
  Secara teori, satu nilai BYTEA dapat mencapai hingga **1 GB**. Namun, ada batasan protokol sekitar **2 GiB** (termasuk header) ketika data diambil melalui `SELECT`.

- **Tidak memiliki encoding atau collation**
  BYTEA tidak mengenal konsep encoding (UTF-8, ASCII, dan sebagainya) maupun collation. Data disimpan sebagai **raw bytes**, sehingga PostgreSQL tidak melakukan interpretasi karakter apa pun.

- **TOAST-enabled**
  Untuk data berukuran besar, PostgreSQL secara otomatis memanfaatkan mekanisme **TOAST (The Oversized-Attribute Storage Technique)**. Ini memungkinkan data besar disimpan dan dikelola secara efisien di luar halaman utama tabel.

- **Perbandingan berbasis byte sequence**
  Operasi perbandingan (`=`, `<`, `>`) pada BYTEA dilakukan berdasarkan urutan byte, bukan berdasarkan makna karakter seperti pada tipe teks.

- **Overhead penyimpanan**
  Di level penyimpanan disk:

  - Data < 127 byte memiliki overhead 1 byte
  - Data ≥ 127 byte memiliki overhead 4 byte
    Overhead ini adalah metadata internal yang digunakan PostgreSQL untuk mengelola panjang data.

#### Contoh Penggunaan BYTEA

Berikut contoh sederhana penggunaan BYTEA dalam sebuah tabel PostgreSQL:

```sql
-- Membuat table dengan BYTEA column
CREATE TABLE bytea_example (
    file_name TEXT,
    data BYTEA
);
```

Tabel di atas memiliki kolom `data` bertipe BYTEA yang akan menyimpan data biner.

Untuk memasukkan data ke kolom BYTEA, salah satu cara yang umum adalah menggunakan **representasi hex**:

```sql
-- Insert data sebagai hex (format: \x + hex digits)
INSERT INTO bytea_example VALUES (
    'hello.txt',
    '\x48656c6c6f20576f726c64'  -- "Hello World" dalam hex
);
```

Pada contoh ini:

- Prefix `\x` menandakan bahwa data yang diberikan adalah dalam format hexadecimal.
- Deretan angka dan huruf setelahnya merepresentasikan byte-byte data biner.

Untuk melihat data yang telah disimpan:

```sql
-- View data
SELECT * FROM bytea_example;
```

Hasil `SELECT` bisa berbeda tergantung client atau tool yang digunakan. Beberapa client akan menampilkan data dalam bentuk hex, sementara yang lain mungkin menampilkan representasi biner tertentu.

#### Hex Representation

Agar lebih jelas, mari kita lihat bagaimana string `"Hello World"` direpresentasikan dalam bentuk byte dan hex:

```
String: "Hello World"
Bytes:  48 65 6c 6c 6f 20 57 6f 72 6c 64
Hex:    \x48656c6c6f20576f726c64
```

Setiap karakter memiliki nilai byte tertentu:

- `H` = `0x48` = 72 (desimal)
- `e` = `0x65` = 101
- `l` = `0x6c` = 108
- dan seterusnya hingga seluruh string direpresentasikan sebagai rangkaian byte.

Dengan memahami konsep ini, dapat terlihat bahwa BYTEA tidak menyimpan teks sebagai “huruf”, melainkan sebagai **angka byte**. Inilah yang membuat BYTEA fleksibel dan aman untuk menyimpan data biner apa pun tanpa risiko perubahan encoding atau interpretasi karakter.

### b. BYTEA vs TEXT

Perbedaan antara `BYTEA` dan `TEXT` di PostgreSQL bukan sekadar soal tipe data, tetapi menyangkut **cara database memahami, menyimpan, dan membandingkan isi data tersebut**. Memahami perbedaan ini penting agar kita tidak salah memilih tipe data, terutama saat berhadapan dengan data biner atau data yang ditujukan untuk dibaca manusia.

#### Perbedaan Konsep Dasar

Secara fundamental, `TEXT` dan `BYTEA` dirancang untuk kebutuhan yang berbeda. Ringkasannya dapat dilihat dari beberapa aspek utama berikut:

- **Content**
  `TEXT` menyimpan data sebagai karakter (huruf, angka, simbol) yang memiliki makna bahasa.
  `BYTEA` menyimpan data sebagai raw bytes, yaitu angka-angka byte tanpa arti linguistik.

- **Encoding**
  `TEXT` selalu terikat pada encoding tertentu, seperti UTF-8 atau Latin1. PostgreSQL perlu tahu encoding agar bisa menafsirkan dan memproses karakter dengan benar.
  `BYTEA` tidak memiliki encoding sama sekali. Data disimpan apa adanya, byte demi byte.

- **Collation**
  Pada `TEXT`, collation menentukan aturan perbandingan dan pengurutan string (misalnya apakah huruf besar dan kecil dianggap sama).
  `BYTEA` tidak memiliki collation, karena tidak ada konsep “huruf” atau “bahasa”.

- **Comparison**
  Perbandingan `TEXT` mengikuti aturan karakter dan collation.
  Perbandingan `BYTEA` dilakukan secara byte-by-byte, sehingga setiap perbedaan satu byte saja akan dianggap berbeda.

- **Use case**
  `TEXT` cocok untuk data yang akan dibaca atau ditulis oleh manusia.
  `BYTEA` cocok untuk data mesin, seperti file biner, hasil hash, token, atau data terenkripsi.

- **Overhead penyimpanan**
  Keduanya menggunakan mekanisme varlena PostgreSQL, dengan overhead internal sekitar 1–4 byte tergantung ukuran data.

#### Contoh Definisi Tabel

Perbedaan ini dapat terlihat sejak tahap pembuatan tabel.

```sql
-- TEXT: Menyimpan dengan encoding dan collation
CREATE TABLE text_table (
    content TEXT  -- Stored with UTF-8 encoding
);

-- BYTEA: Menyimpan raw bytes
CREATE TABLE binary_table (
    content BYTEA  -- Stored as-is, no interpretation
);
```

Pada tabel `text_table`, PostgreSQL akan memastikan bahwa data di kolom `content` sesuai dengan encoding database (misalnya UTF-8). Setiap karakter dipahami sebagai unit teks.

Sebaliknya, pada `binary_table`, kolom `content` hanya berisi deretan byte. PostgreSQL tidak peduli apakah byte tersebut merepresentasikan teks, gambar, atau data acak.

#### Perbedaan Perilaku Saat Comparison

Perbedaan paling sering menimbulkan kejutan adalah saat melakukan perbandingan data.

```sql
-- TEXT: 'abc' vs 'ABC' tergantung collation
-- BYTEA: 0x616263 vs 0x414243 byte-by-byte exact match
```

Pada `TEXT`:

- `'abc'` dan `'ABC'` bisa dianggap sama atau berbeda tergantung collation yang digunakan.
- Dalam collation tertentu, perbandingan bisa bersifat case-insensitive.

Pada `BYTEA`:

- `0x61` (huruf `a`) dan `0x41` (huruf `A`) adalah byte yang berbeda.
- Karena perbandingan dilakukan secara exact byte-by-byte, hasilnya **selalu berbeda**.

#### Kapan Memilih TEXT atau BYTEA

Sebagai aturan praktis:

- Gunakan `TEXT` jika data memang berupa teks yang akan ditampilkan, dicari, atau dibandingkan berdasarkan aturan bahasa.
- Gunakan `BYTEA` jika data tidak boleh diinterpretasikan sebagai teks, atau jika Anda membutuhkan keakuratan byte secara mutlak.

Dengan memahami perbedaan ini, pemilihan antara `TEXT` dan `BYTEA` menjadi lebih jelas dan sesuai dengan tujuan desain database yang baik.

### c. BYTEA Output Formats

Saat bekerja dengan kolom `BYTEA`, penting untuk memahami bahwa PostgreSQL menyediakan **dua format output** yang berbeda untuk menampilkan data biner. Format ini hanya memengaruhi **cara data ditampilkan ke client**, bukan cara data disimpan di dalam database. Secara internal, data BYTEA tetap sama—yang berubah hanyalah representasinya saat di-query.

#### 1. Hex Format (Default & Recommended)

Secara default, PostgreSQL menggunakan **hex format** untuk menampilkan data BYTEA.

```sql
-- Melihat format output saat ini
SHOW bytea_output;
-- Output: hex (default)

-- Data ditampilkan dalam hexadecimal
SELECT data FROM bytea_example;
-- Output: \x48656c6c6f20576f726c64
```

Pada format ini:

- Prefix `\x` menandakan bahwa data ditampilkan dalam bentuk hexadecimal.
- Setiap byte direpresentasikan oleh **dua digit hex**, sehingga strukturnya konsisten dan mudah dipetakan kembali ke data biner aslinya.

Sebagai contoh, output `\x48656c6c6f20576f726c64` adalah representasi hex dari byte-byte string `"Hello World"`.

##### Keuntungan Hex

Hex format menjadi pilihan yang direkomendasikan karena beberapa alasan penting:

- Tidak ambigu
  Setiap byte selalu direpresentasikan oleh dua digit hex, sehingga tidak ada keraguan dalam interpretasi data.

- Format standar
  Hexadecimal adalah representasi umum untuk data biner dan dikenal luas di berbagai sistem dan bahasa pemrograman.

- Tool-friendly
  Banyak tool, library, dan driver database secara default mengharapkan atau lebih mudah memproses format hex.

- Tidak ada risiko data loss
  Semua kemungkinan nilai byte (0–255) dapat direpresentasikan dengan aman tanpa pengecualian.

#### 2. Escape Format (Legacy)

Selain hex, PostgreSQL juga mendukung **escape format**, yang merupakan format lama (legacy).

```sql
-- Mengubah ke escape format
SET bytea_output = 'escape';

SELECT data FROM bytea_example;
-- Output: Hello World (jika bytes adalah valid ASCII)
-- Atau: \000\001\002... (untuk non-ASCII bytes)
```

Dalam escape format:

- Byte yang merepresentasikan karakter ASCII yang dapat dibaca akan ditampilkan sebagai karakter biasa.
- Byte non-ASCII atau byte kontrol akan ditampilkan menggunakan escape sequence seperti `\000`, `\001`, dan seterusnya.

Sekilas, format ini terlihat “lebih ramah manusia” jika data kebetulan berupa teks ASCII, tetapi justru di sinilah potensi masalah muncul.

##### Masalah Escape Format

Escape format memiliki beberapa kelemahan yang membuatnya tidak direkomendasikan untuk penggunaan modern:

- Tidak semua byte bisa direpresentasikan sebagai karakter
  Data biner murni sering kali mengandung byte yang tidak memiliki padanan karakter yang aman untuk ditampilkan.

- Output bisa membingungkan
  Campuran antara karakter yang bisa dibaca dan escape sequence membuat hasil query sulit dipahami dan rawan salah tafsir.

- Format legacy
  Escape format dipertahankan terutama untuk kompatibilitas ke belakang, bukan untuk penggunaan baru.

#### Perbandingan Hex vs Escape

Perbedaan kedua format ini akan sangat jelas saat menangani data biner nyata, misalnya header file gambar:

```sql
-- Data: Binary image header
-- Hex format:    \x89504e470d0a1a0a  ✅ Jelas
-- Escape format: \211PNG\r\n\032\n  ⚠️ Campuran readable/escape
```

Pada hex format, seluruh data ditampilkan secara konsisten dan eksplisit.
Pada escape format, sebagian byte muncul sebagai karakter (`PNG`), sementara yang lain sebagai escape (`\211`, `\r`, `\n`), sehingga sulit dibaca dan kurang aman untuk pemrosesan lanjutan.

Karena alasan-alasan tersebut, **hex format adalah pilihan default dan yang paling disarankan** saat bekerja dengan BYTEA di PostgreSQL, terutama untuk sistem modern dan integrasi dengan berbagai tool.

### d. TOAST untuk Binary Data

Seperti halnya tipe `TEXT`, kolom bertipe `BYTEA` juga memanfaatkan mekanisme **TOAST (The Oversized-Attribute Storage Technique)** ketika menyimpan data berukuran besar. TOAST adalah fitur internal PostgreSQL yang dirancang untuk menjaga agar baris tabel tetap efisien, meskipun menyimpan data yang ukurannya bisa sangat besar.

#### Cara Kerja TOAST dengan BYTEA

Untuk memahami perannya, mari lihat contoh struktur tabel berikut:

```sql
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    name TEXT,
    content BYTEA  -- Bisa sampai 1GB
);
```

Kolom `content` di sini berpotensi menyimpan data biner besar, seperti file gambar, dokumen, atau hasil kompresi.

#### Penyimpanan Data Kecil

```sql
-- Insert file kecil (10 KB)
INSERT INTO files VALUES (1, 'small.jpg', '\x...');
-- Disimpan inline di row
```

Jika data yang dimasukkan relatif kecil, PostgreSQL akan menyimpannya **langsung di dalam baris tabel (inline)**. Ini berarti seluruh data `BYTEA` berada di halaman data yang sama dengan kolom lainnya. Untuk ukuran kecil, pendekatan ini paling efisien karena tidak memerlukan akses tambahan.

#### Penyimpanan Data Besar dan TOAST

```sql
-- Insert file besar (500 KB)
INSERT INTO files VALUES (2, 'large.jpg', '\x...');
-- Otomatis di-TOAST:
-- - Row hanya simpan pointer
-- - Actual data di TOAST table
-- - Row tetap compact ✅
```

Ketika ukuran data melewati ambang batas tertentu, PostgreSQL secara otomatis mengaktifkan TOAST:

- Baris utama tabel hanya menyimpan **pointer** ke data.
- Data biner yang sebenarnya dipindahkan ke **TOAST table** internal.
- Ukuran baris utama tetap kecil dan stabil, sehingga tidak membebani penyimpanan dan indeks.

Proses ini sepenuhnya transparan bagi pengguna. Dari sudut pandang SQL, tidak ada perbedaan cara mengakses data, baik data tersebut disimpan inline maupun di-TOAST.

#### Dampak TOAST pada Performa Query

Salah satu keuntungan terbesar TOAST adalah efisiensi saat melakukan query.

```sql
-- Query tanpa content column
SELECT id, name FROM files;
-- Super cepat! Tidak perlu akses TOAST table
```

Jika query tidak menyertakan kolom `content`, PostgreSQL **tidak perlu mengambil data dari TOAST table**. Akibatnya, query seperti ini berjalan sangat cepat karena hanya membaca data ringan dari tabel utama.

```sql
-- Query dengan content column
SELECT * FROM files WHERE id = 2;
-- Fetch dari TOAST table saat dibutuhkan
```

Sebaliknya, ketika kolom `content` diminta, PostgreSQL baru akan mengambil data dari TOAST table. Akses tambahan ini hanya terjadi saat benar-benar dibutuhkan, sehingga beban sistem tetap terkontrol.

#### Ringkasan Peran TOAST pada BYTEA

Dengan TOAST, PostgreSQL mampu:

- Menyimpan data biner berukuran besar tanpa membuat baris tabel menjadi gemuk.
- Mengoptimalkan query yang tidak membutuhkan data biner.
- Menyediakan mekanisme penyimpanan besar yang efisien dan transparan.

Inilah alasan mengapa `BYTEA` dapat digunakan dengan aman untuk menyimpan binary data besar, tanpa harus mengorbankan performa query sehari-hari.

### e. Kapan (Tidak) Menggunakan BYTEA untuk Files

Topik penyimpanan file di database sering memicu perdebatan, dan materi ini mengambil posisi yang cukup tegas. Intinya, **BYTEA bukan solusi umum untuk menyimpan file berukuran besar**, meskipun secara teknis PostgreSQL mampu melakukannya. Memahami alasan di balik rekomendasi ini akan membantu kita membuat keputusan arsitektur yang lebih sehat.

#### ❌ Tidak Direkomendasikan: File Besar di Database

```sql
-- ❌ JANGAN: Store large files di database
CREATE TABLE user_uploads (
    user_id INTEGER,
    file_name TEXT,
    file_data BYTEA  -- 50MB per file
);
```

Pendekatan ini terlihat sederhana: semua data, termasuk file, berada di satu tempat. Namun, dalam praktiknya, cara ini menimbulkan banyak masalah serius:

- **Database cepat membengkak**
  File berukuran puluhan megabyte akan membuat ukuran database tumbuh sangat cepat, bahkan jika jumlah user masih sedikit.

- **Backup menjadi lambat dan berat**
  Setiap backup harus menyalin seluruh data biner tersebut, sehingga proses backup memakan waktu dan storage ekstra.

- **Overhead replication tinggi**
  Pada sistem dengan replication, setiap perubahan file akan direplikasi ke semua node, meningkatkan beban jaringan dan storage.

- **Tekanan memori saat query**
  Query yang tidak sengaja menarik kolom `BYTEA` besar dapat mengonsumsi memori dalam jumlah besar dan menurunkan performa.

- **Tidak bisa memanfaatkan CDN**
  Database tidak dirancang untuk menjadi file server, sehingga Anda kehilangan keuntungan distribusi global dan caching dari CDN.

Karena alasan-alasan ini, menyimpan file besar langsung di database umumnya dianggap sebagai **anti-pattern**.

#### ✅ Direkomendasikan: File Storage + Pointer di Database

Pendekatan yang lebih sehat adalah memisahkan tanggung jawab antara database dan storage file.

```sql
-- ✅ LAKUKAN: Store file di S3/R2/Tigris, pointer di DB
CREATE TABLE user_uploads (
    user_id INTEGER,
    file_name TEXT,
    file_url TEXT,  -- https://s3.amazonaws.com/bucket/file.jpg
    file_size INTEGER,
    uploaded_at TIMESTAMP
);
```

Pada desain ini:

- File disimpan di layanan object storage seperti S3, R2, atau Tigris.
- Database hanya menyimpan metadata dan **pointer** (URL atau key) ke file tersebut.

Keuntungan utama pendekatan ini antara lain:

- **Database tetap kecil dan cepat**
  Hanya metadata yang disimpan, bukan data biner besar.

- **Mudah menggunakan CDN**
  File dapat didistribusikan secara global dengan latensi rendah.

- **Backup lebih cepat**
  Backup database menjadi ringan karena tidak membawa file besar.

- **Skalabilitas lebih fleksibel**
  Database dan file storage dapat diskalakan secara independen sesuai kebutuhan.

#### Trade-off Penyimpanan Database vs File Storage

Tidak ada solusi yang benar-benar “sempurna”. Setiap pendekatan memiliki kelebihan dan kekurangan:

```
Database Storage
├─ Pro: Strong consistency, ACID guarantees
├─ Pro: Single source of truth
└─ Con: Not designed for large binary blobs

File Storage (S3, etc)
├─ Pro: Optimized untuk files
├─ Pro: CDN integration
├─ Pro: Cheaper per GB
└─ Con: Eventually consistent, bisa out-of-sync dengan DB
```

Database unggul dalam konsistensi dan transaksi, sedangkan file storage unggul dalam efisiensi, biaya, dan distribusi file.

#### ⚖️ Mungkin Masuk Akal: File Sangat Kecil

Ada kasus tertentu di mana menggunakan BYTEA masih bisa diterima.

```sql
-- Mungkin OK untuk files sangat kecil (< 100KB)
CREATE TABLE user_profiles (
    user_id INTEGER,
    avatar BYTEA  -- Small thumbnail (50KB)
);
```

Untuk file yang benar-benar kecil, seperti thumbnail avatar:

- **Konsistensi kuat**
  Data avatar dan data user bisa disimpan dalam satu transaksi atomik.

- **Sederhana secara operasional**
  Tidak perlu mengelola kredensial atau lifecycle object storage.

- **Atomicity terjamin**
  Tidak ada risiko data user ter-update tetapi file gagal tersimpan, atau sebaliknya.

Namun, tetap ada pertanyaan yang perlu dijawab sebelum memilih pendekatan ini:

- Apakah konsistensi sekuat ini benar-benar dibutuhkan?
- Apakah ukuran file akan tetap kecil di masa depan?
- Apakah aplikasi membutuhkan CDN untuk performa?

#### Managed Providers dan Pendekatan Hybrid

Beberapa managed database provider menawarkan solusi hybrid.

```sql
-- Beberapa provider handle ini otomatis
-- File > threshold (misal 5MB) → auto-moved ke S3 + CDN
-- User tidak perlu manage
CREATE TABLE files (
    content BYTEA  -- Provider handle TOAST → S3 transition
);
```

Pada skenario ini, developer tetap menggunakan `BYTEA`, tetapi provider secara otomatis:

- Menyimpan data kecil di database.
- Memindahkan data besar ke object storage + CDN di belakang layar.

Pendekatan ini mengurangi kompleksitas operasional, tetapi tetap penting untuk memahami apa yang sebenarnya terjadi di balik layar agar desain sistem tetap terkontrol dan tidak bergantung secara membabi buta pada abstraksi provider.

### f. Use Case Valid: Checksums dan Hashes

Binary data, khususnya `BYTEA`, sangat cocok digunakan untuk menyimpan **hash** dan **checksum**. Pada konteks ini, yang disimpan bukanlah data aslinya, melainkan hasil transformasi berupa deretan byte yang merepresentasikan sidik jari (fingerprint) dari suatu input. Pendekatan ini umum digunakan untuk verifikasi integritas data, deduplikasi, hingga kebutuhan keamanan—dengan catatan algoritma yang dipilih harus sesuai dengan tujuannya.

#### MD5 Hash (⚠️ Not Secure!)

PostgreSQL menyediakan fungsi `MD5()` secara bawaan untuk menghasilkan hash MD5 dari sebuah string.

```sql
-- Generate MD5 hash
SELECT MD5('Hello World');
-- Output: 3e25960a79dbc69b674cd4ec67a72c62 (TEXT)
```

Hasil MD5 di PostgreSQL berupa **string hex**, bukan binary. Ini bisa dikonfirmasi dengan mengecek tipe datanya:

```sql
-- Cek tipe data
SELECT pg_typeof(MD5('Hello World'));
-- Output: text
```

Artinya, meskipun MD5 secara konsep adalah hash biner 128-bit, PostgreSQL menampilkannya sebagai `TEXT` yang berisi representasi hexadecimal.

##### Peringatan Keamanan MD5

MD5 **tidak aman secara kriptografis** dan sudah lama dianggap rusak.

```
MD5 Status: CRYPTOGRAPHICALLY BROKEN ❌
```

MD5 rentan terhadap:

- **Collision attacks**: dua input berbeda bisa menghasilkan hash yang sama.
- **Preimage attacks**: dengan teknik tertentu, hash bisa ditelusuri kembali ke input.
- Sudah digunakan dan dianalisis selama puluhan tahun, sehingga kelemahannya dipahami dengan sangat baik.

Karena itu, **jangan pernah** menggunakan MD5 untuk:

- Password hashing
- Operasi keamanan atau kriptografi
- Apa pun yang bersifat security-critical

Namun, MD5 masih **boleh digunakan** untuk kebutuhan non-keamanan, seperti:

- Checksums untuk mendeteksi korupsi data (bukan manipulasi)
- Hash cepat untuk cache key
- Deduplication sederhana
- Kasus di mana kecepatan lebih penting daripada keamanan

#### SHA-256 (Alternatif yang Aman)

Untuk kebutuhan yang benar-benar terkait keamanan, PostgreSQL menyediakan fungsi hashing yang lebih kuat melalui extension `pgcrypto`.

```sql
-- SHA-256 hash (requires pgcrypto extension)
SELECT digest('Hello World', 'sha256');
-- Output: \xa591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e
```

Berbeda dengan MD5, fungsi `digest()`:

- Menghasilkan output **binary murni**
- Mengembalikan tipe data `BYTEA`

Hal ini bisa dilihat dengan jelas dari pengecekan tipe datanya:

```sql
-- Cek tipe data
SELECT pg_typeof(digest('Hello World', 'sha256'));
-- Output: bytea
```

Karena hasilnya berupa `BYTEA`, SHA-256 sangat cocok disimpan sebagai binary data tanpa perlu konversi ke teks. Selain itu, SHA-256 saat ini dianggap **aman secara kriptografis** untuk banyak use case modern.

SHA-256 sebaiknya digunakan untuk:

- Operasi yang bersifat security-critical
- Verifikasi integritas data yang membutuhkan perlindungan dari manipulasi
- Sistem yang mengandalkan hash sebagai bagian dari mekanisme keamanan

Untuk password hashing secara khusus, algoritma yang lebih tepat adalah:

- `bcrypt`
- `argon2`
  karena keduanya dirancang tahan terhadap brute-force dan serangan modern.

#### Perbedaan Tipe Return: MD5 vs SHA-256

Ada satu keanehan kecil di PostgreSQL yang sering membingungkan di awal:

```sql
-- MD5 returns TEXT
MD5('data') → '3e25960...' (TEXT)

-- SHA-256 returns BYTEA
digest('data', 'sha256') → \xa591... (BYTEA)
```

MD5 mengembalikan `TEXT` berupa hex string, sedangkan SHA-256 melalui `digest()` mengembalikan `BYTEA`. Secara desain ini memang tidak konsisten, tetapi itulah perilaku bawaan PostgreSQL.

Justru dari sini terlihat dengan jelas kapan `BYTEA` menjadi pilihan yang tepat: **saat kita ingin menyimpan data biner asli**, seperti hasil hash kriptografis, tanpa kehilangan informasi atau melakukan encoding tambahan.

### g. Converting MD5 to BYTEA

Secara default, PostgreSQL mengembalikan hasil `MD5()` dalam bentuk **TEXT** berupa string hexadecimal. Meskipun mudah dibaca, format ini sebenarnya tidak efisien dari sisi storage maupun performa. Untuk kebutuhan seperti checksum, deduplikasi, atau indexing hash, mengonversi hasil MD5 ke **BYTEA** adalah praktik yang lebih baik.

#### MD5 dalam Bentuk TEXT (Default)

```sql
-- MD5 sebagai text (default)
SELECT MD5('Hello World');
-- Output: 3e25960a79dbc69b674cd4ec67a72c62 (32 chars TEXT)
```

Output di atas terdiri dari 32 karakter hexadecimal. Setiap dua karakter hex merepresentasikan satu byte data biner, sehingga secara logika hash MD5 ini sebenarnya hanya berukuran **16 byte**, meskipun tampilannya terlihat lebih panjang.

#### Mengonversi MD5 ke BYTEA

PostgreSQL menyediakan fungsi `decode()` untuk mengubah string hex menjadi data biner.

```sql
-- Convert ke BYTEA dengan decode
SELECT decode(MD5('Hello World'), 'hex');
-- Output: \x3e25960a79dbc69b674cd4ec67a72c62 (16 bytes BYTEA)
```

Pada hasil ini:

- Prefix `\x` menunjukkan representasi hex dari data `BYTEA`.
- Panjang data biner yang sebenarnya adalah **16 byte**, sesuai dengan spesifikasi MD5 (128-bit).

#### Perbandingan Ukuran Storage

Untuk melihat perbedaan ukuran secara nyata, kita bisa menggunakan `pg_column_size()`.

```sql
-- Perbandingan ukuran (VERIFIED!) ✅
SELECT
    pg_column_size(MD5('Hello World')) AS md5_text_size,
    pg_column_size(decode(MD5('Hello World'), 'hex')) AS md5_bytea_size;
```

Hasilnya:

- `md5_text_size`: **36 bytes**
  Terdiri dari 32 byte data (karakter hex) + 4 byte overhead varlena untuk `TEXT`.

- `md5_bytea_size`: **20 bytes**
  Terdiri dari 16 byte data biner + 4 byte overhead varlena untuk `BYTEA`.

Perbedaan ini menunjukkan penghematan sekitar **44%**, yang akan sangat signifikan jika hash disimpan dalam jumlah besar.

#### Mengapa Perlu Mengonversi ke BYTEA

Ada beberapa alasan kuat mengapa konversi ini direkomendasikan:

1. **Ukuran lebih kecil**
   20 byte dibanding 36 byte berarti penggunaan storage jauh lebih efisien.

2. **Perbandingan lebih cepat**
   PostgreSQL membandingkan `BYTEA` secara byte-by-byte, yang umumnya lebih cepat daripada perbandingan karakter pada `TEXT`.

3. **Tanpa collation overhead**
   `BYTEA` tidak memiliki encoding maupun collation, sehingga tidak ada biaya tambahan saat membandingkan atau mengindeks data.

4. **Ukuran data lebih prediktif**
   MD5 dalam bentuk biner selalu memiliki payload tetap 16 byte, sehingga lebih konsisten untuk desain indeks dan estimasi ukuran.

#### Penjelasan Storage Overhead

Fungsi `pg_column_size()` mengukur **total ukuran storage**, termasuk overhead internal PostgreSQL.

```
MD5 as TEXT (32 hex chars):
- Data: 32 bytes
- Varlena overhead: 4 bytes
- Total: 36 bytes

MD5 as BYTEA (16 bytes binary):
- Data: 16 bytes
- Varlena overhead: 4 bytes
- Total: 20 bytes
```

Perlu dicatat bahwa:

- Overhead penyimpanan di disk bisa berbeda (1 byte untuk data < 127 byte, 4 byte untuk data ≥ 127 byte).
- Namun `pg_column_size()` menampilkan ukuran aktual yang digunakan PostgreSQL, termasuk padding dan metadata internal, sehingga angka ini bisa dijadikan referensi yang andal.

#### Alur Proses Konversi

Untuk merangkum prosesnya secara konseptual:

```
'Hello World'
    ↓ MD5()
'3e25960a79dbc69b674cd4ec67a72c62' (32 hex chars sebagai TEXT)
    ↓ decode(..., 'hex')
\x3e25960a79dbc69b674cd4ec67a72c62 (16 bytes sebagai BYTEA)
```

Dengan pendekatan ini, kita tetap mendapatkan hash MD5 yang sama secara logis, tetapi dalam bentuk yang **lebih efisien, lebih cepat, dan lebih sesuai** untuk penyimpanan serta pemrosesan di level database.

### h. UUID: Best Balance of Ergonomics & Efficiency

PostgreSQL menyediakan tipe data `UUID` yang sangat optimal untuk menyimpan **identifier berbasis hash**, termasuk hash 128-bit seperti MD5. UUID menawarkan kombinasi yang jarang: **ukuran penyimpanan paling kecil, performa tinggi, dan tetap nyaman dibaca manusia**. Inilah alasan mengapa UUID sering dianggap sebagai titik keseimbangan terbaik antara ergonomi dan efisiensi.

#### Mengonversi MD5 ke UUID

Hasil MD5 secara alami berukuran 128-bit, yang kebetulan **persis sama dengan ukuran UUID**. Karena itu, MD5 dapat langsung di-cast ke UUID.

```sql
-- Cast MD5 ke UUID
SELECT MD5('Hello World')::UUID;
-- Output: 3e25960a-79db-c69b-674c-d4ec67a72c62
```

Perhatikan bahwa:

- Nilai hash yang sama tetap dipertahankan.
- UUID menambahkan **tanda dash (`-`)** untuk meningkatkan keterbacaan.
- Ini hanyalah perubahan format tampilan, bukan perubahan isi data.

#### Perbandingan Ukuran Storage

Untuk melihat perbedaan nyata antar tipe data, berikut perbandingan ukuran penyimpanan menggunakan `pg_column_size()`:

```sql
-- Perbandingan ukuran storage (VERIFIED!) ✅
SELECT
    pg_column_size(MD5('Hello World')) AS text_size,
    pg_column_size(decode(MD5('Hello World'), 'hex')) AS bytea_size,
    pg_column_size(MD5('Hello World')::UUID) AS uuid_size;
```

Hasilnya:

- `TEXT`: **36 bytes**
- `BYTEA`: **20 bytes**
- `UUID`: **16 bytes** ✅ paling kecil

UUID menjadi yang paling efisien karena **tidak menggunakan varlena overhead** sama sekali.

#### Struktur dan Format UUID

UUID hanyalah cara terstruktur untuk menampilkan data 128-bit.

```
MD5 output:     3e25960a79dbc69b674cd4ec67a72c62
UUID format:    3e25960a-79db-c69b-674c-d4ec67a72c62
                └──┬───┘ └─┬┘ └─┬┘ └─┬┘ └────┬─────┘
                   │       │    │    │       │
                8 digits   4    4    4    12 digits
```

Secara internal:

- UUID disimpan sebagai **16 byte (128-bit)**.
- Ukurannya **tetap (fixed-size)**.
- Tidak ada overhead tambahan seperti pada `TEXT` atau `BYTEA`.

Untuk tampilan ke user:

- Data diformat dengan dash agar lebih mudah dibaca.
- Format ini merupakan standar yang diakui luas.

#### Mengapa UUID Menjadi Pilihan Terbaik

Contoh skema tabel berikut memperlihatkan perbandingan langsung antar pendekatan:

```sql
CREATE TABLE file_checksums (
    file_id INTEGER,
    checksum_text TEXT,           -- 36 bytes
    checksum_bytea BYTEA,         -- 20 bytes
    checksum_uuid UUID            -- 16 bytes ✅ BEST!
);
```

Keunggulan UUID dalam konteks ini:

- **Ukuran paling kecil**
  16 byte total, tanpa overhead tambahan.

- **Tipe native PostgreSQL**
  Optimizer dan planner PostgreSQL memahami UUID dengan sangat baik.

- **Fixed-size dan prediktif**
  Sangat ideal untuk indexing dan estimasi performa.

- **Readable untuk manusia**
  Format dengan dash membuat nilai hash lebih mudah dikenali dan dibedakan.

- **Format standar**
  Mengikuti spesifikasi RFC 4122 dan RFC 9562.

- **Ekosistem luas**
  Banyak tool dan library sudah mendukung UUID secara default.

- **Perbandingan cepat**
  Perbandingan dilakukan pada blok tetap 16 byte, tanpa aturan collation.

Sebagai perbandingan singkat:

- `BYTEA`
  Fleksibel untuk berbagai algoritma hash dan data biner, tetapi memiliki overhead varlena dan panjang variabel.

- `TEXT`
  Mudah dibaca tanpa konversi, tetapi paling boros storage dan paling lambat untuk perbandingan.

#### Insight Kunci: UUID adalah Fixed-Size Type

Keunggulan utama UUID terletak pada sifatnya sebagai **tipe fixed-size**:

```
UUID:  16 bytes data + 0 bytes overhead = 16 bytes total ✅
BYTEA: 16 bytes data + 4 bytes overhead = 20 bytes total
TEXT:  32 bytes data + 4 bytes overhead = 36 bytes total
```

Inilah alasan mengapa UUID bisa lebih kecil daripada `BYTEA`, meskipun keduanya sama-sama menyimpan 16 byte data. Untuk hash identifier berukuran tetap, UUID memberikan kombinasi terbaik antara **efisiensi penyimpanan, performa, dan kenyamanan penggunaan**.

### i. Practical Use Case: Content Deduplication

Salah satu use case paling praktis dari hash dan tipe data efisien seperti `UUID` adalah **content deduplication**, yaitu mendeteksi apakah sebuah file atau konten sudah pernah disimpan sebelumnya. Pendekatan ini sangat berguna untuk menghemat storage dan mencegah penyimpanan data yang sama berulang kali.

#### Desain Tabel untuk Deduplication

Berikut contoh desain tabel yang memanfaatkan hash untuk mendeteksi duplikasi konten:

```sql
-- Table untuk store files dengan deduplication
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    file_name TEXT,
    content BYTEA,
    content_hash UUID GENERATED ALWAYS AS (MD5(content)::UUID) STORED
);
```

Beberapa hal penting dari desain ini:

- Kolom `content` menyimpan data biner file.
- Kolom `content_hash` adalah **generated column** yang nilainya dihitung otomatis oleh PostgreSQL.
- Hash dihitung menggunakan `MD5(content)` lalu langsung di-cast ke `UUID`, sehingga:

  - Ukurannya efisien (16 bytes).
  - Nilainya konsisten dan tidak bisa dimodifikasi manual.

- Kata kunci `STORED` berarti hasil hash benar-benar disimpan di tabel, bukan dihitung ulang setiap kali query.

#### Index untuk Pencarian Cepat

Agar pengecekan duplikasi berjalan cepat meskipun data sudah banyak, hash perlu diindeks.

```sql
-- Index pada hash untuk fast lookup
CREATE INDEX idx_files_hash ON files(content_hash);
```

Dengan index ini, PostgreSQL dapat melakukan pencarian berdasarkan hash dalam waktu yang sangat singkat, tanpa perlu melakukan scan seluruh tabel.

#### Menyimpan File Baru

Contoh insert data file ke tabel:

```sql
-- Insert file
INSERT INTO files (file_name, content)
VALUES ('doc1.txt', '\x48656c6c6f');
```

Saat perintah ini dijalankan:

- PostgreSQL otomatis menghitung `MD5(content)`.
- Hasilnya langsung dikonversi ke `UUID`.
- Nilai tersebut disimpan di kolom `content_hash` tanpa perlu ditulis secara eksplisit.

#### Mengecek Duplikasi Sebelum Insert

Sebelum menyimpan file baru, kita bisa mengecek apakah konten yang sama sudah ada di database.

```sql
-- Check for duplicates sebelum insert
WITH new_file AS (
    SELECT '\x48656c6c6f'::BYTEA AS content
)
SELECT f.*
FROM files f, new_file nf
WHERE f.content_hash = MD5(nf.content)::UUID;
```

Penjelasan alurnya:

- CTE `new_file` mensimulasikan file yang baru di-upload.
- Hash dari konten baru dihitung dengan `MD5(nf.content)::UUID`.
- Database mencari baris di tabel `files` dengan `content_hash` yang sama.
- Jika query ini mengembalikan baris, berarti **konten sudah ada** dan file tersebut merupakan duplikat.

#### Alur Deduplication Secara Keseluruhan

Secara konseptual, proses deduplikasi berjalan seperti berikut:

```
User uploads file
    ↓
Calculate MD5 hash
    ↓
Check if hash exists in DB
    ↓
├─ Exists → Show "File already uploaded"
└─ Not exists → Insert new file
```

Dengan pendekatan ini:

- Proses pengecekan sangat cepat karena berbasis index.
- Storage menjadi lebih efisien karena file duplikat tidak disimpan ulang.
- Logika aplikasi menjadi sederhana, karena database membantu menjaga konsistensi deduplikasi.

Meskipun MD5 digunakan di sini, konteksnya adalah **deduplikasi dan deteksi kesamaan konten**, bukan keamanan. Untuk use case ini, MD5 sudah cukup cepat dan praktis, selama dipahami bahwa tujuannya adalah mendeteksi duplikasi, bukan mencegah manipulasi data.

## 3. Hubungan Antar Konsep

### Storage Efficiency Hierarchy (VERIFIED!):

```
Most Efficient
    ↑
UUID (MD5 cast)          16 bytes  ✅ SMALLEST! (fixed-size, no overhead)
    ↑
BYTEA (decoded hex)      20 bytes  ✅ Flexible (varlena overhead)
    ↑
TEXT (hex string)        36 bytes  ⚠️ WASTEFUL (readable but big)
    ↑
Least Efficient
```

### Data Type Decision Tree:

```
Apa yang mau disimpan?

├─ Human-readable text
│   └─ TEXT (dengan encoding & collation)
│
├─ Machine data / bytes
│   ├─ Small (< 1KB): BYTEA ✅
│   ├─ Medium (1KB-1MB): BYTEA dengan consideration
│   └─ Large (> 1MB): File storage + pointer ✅
│
├─ Hash / Checksum (MD5, SHA, etc)
│   ├─ Best choice: UUID ✅ (16 bytes, native type)
│   ├─ Flexible: BYTEA ✅ (20 bytes, works with any hash)
│   └─ Human-readable: TEXT (36 bytes, wasteful)
│
└─ Files
    ├─ Very small (< 100KB): Maybe BYTEA
    └─ Anything larger: S3/R2 + URL pointer ✅
```

### Why UUID Wins for Hashes:

```
UUID advantages over BYTEA for MD5/hash storage:

1. Smaller: 16 bytes vs 20 bytes (20% space saving)
2. Faster: Fixed-size comparison vs variable-length
3. Indexing: Better index performance
4. Standard: RFC-compliant UUID type
5. Tooling: Better ecosystem support
6. Readable: Formatted output with dashes

Use BYTEA when:
- Need flexibility (SHA-256, SHA-512, custom hashes)
- Storing non-hash binary data
- Hash size varies

Use UUID when:
- Storing 128-bit hashes (MD5, etc)
- Want smallest storage
- Want best performance
```

### TOAST Usage:

```
TEXT column
    └─ Large text (> ~2KB) → TOAST
         └─ Same mechanism

BYTEA column
    └─ Large binary (> ~2KB) → TOAST
         └─ Keeps rows compact
              └─ Fetch only when needed
```

## 4. Kesimpulan

Binary data di PostgreSQL disimpan menggunakan tipe BYTEA, yang menyimpan raw bytes tanpa encoding atau collation. Untuk hash identifiers (seperti MD5), UUID type adalah pilihan terbaik karena memberikan storage paling kecil (16 bytes) tanpa varlena overhead.

**Key Points:**

1. **BYTEA**: Variable-length binary, max 1GB (protocol limit 2GB untuk SELECT)
2. **Output Formats**: Hex (recommended) vs Escape (legacy)
3. **TOAST**: Large binary data otomatis di-TOAST
4. **File Storage**: Large files → S3/R2 + pointer, NOT in database
5. **Hashes**: MD5 (not secure!) vs SHA-256 (secure)
6. **Storage Sizes**: UUID (16B) < BYTEA (20B) < TEXT (36B)

**Best Practices:**

- ✅ Gunakan UUID untuk MD5/128-bit hashes (smallest & fastest!)
- ✅ Gunakan BYTEA untuk flexibility (SHA-256, arbitrary binary)
- ✅ Store large files di file storage (S3, R2, Tigris)
- ✅ Simpan metadata dan pointers di database
- ✅ Set `bytea_output = 'hex'` untuk consistency
- ❌ Jangan store files > 1MB di database
- ❌ Jangan gunakan MD5 untuk security (use SHA-256+)
- ❌ Jangan gunakan TEXT untuk hashes (wasteful!)

**Storage Efficiency untuk MD5 Hashes (VERIFIED!):**

```
UUID (MD5 cast):          16 bytes  ✅ SMALLEST & BEST!
    - Fixed-size type
    - No varlena overhead
    - Native PostgreSQL optimization
    - Best index performance

BYTEA (decoded):          20 bytes  ✅ Flexible
    - Variable-length type
    - 4 bytes varlena overhead
    - Works with any hash size

TEXT (hex string):        36 bytes  ❌ WASTEFUL
    - Variable-length type
    - 4 bytes varlena overhead
    - 32 bytes for hex representation
    - Only use when human-readability is critical
```
