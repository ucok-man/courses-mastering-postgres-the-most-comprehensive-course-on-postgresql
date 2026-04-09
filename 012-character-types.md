# 📝 Tipe Data Karakter di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tiga tipe data untuk menyimpan karakter di PostgreSQL: `CHAR` (fixed-length), `VARCHAR` (character varying), dan `TEXT`. Yang mengejutkan, ketiga tipe ini menggunakan struktur penyimpanan yang identik di balik layar, sehingga performa mereka sebenarnya sama. Video ini menjelaskan mengapa `CHAR` sebaiknya dihindari, kapan menggunakan `VARCHAR`, dan mengapa `TEXT` menjadi pilihan yang direkomendasikan dalam banyak kasus.

## 2. Konsep Utama

### a. Tiga Tipe Data Karakter

PostgreSQL menyediakan tiga tipe data utama untuk menyimpan karakter atau teks. Secara sintaks, ketiganya terlihat berbeda dan sering diasosiasikan dengan perilaku yang juga berbeda. Namun, ada satu fakta penting yang sering mengejutkan banyak orang: di PostgreSQL, ketiga tipe ini menggunakan struktur penyimpanan internal yang sama.

Tiga tipe data karakter tersebut adalah:

1. **CHAR(n) / CHARACTER(n)** – tipe dengan panjang tetap (fixed-length).
2. **VARCHAR(n) / CHARACTER VARYING(n)** – tipe dengan panjang variabel, tetapi memiliki batas maksimum.
3. **TEXT** – tipe dengan panjang variabel tanpa batas eksplisit.

Meskipun konsepnya berbeda, PostgreSQL memperlakukan ketiganya secara sangat mirip dari sisi penyimpanan. Tidak ada perbedaan signifikan dalam cara data disimpan di disk, sehingga pemilihan tipe ini lebih berkaitan dengan validasi dan semantik data, bukan efisiensi storage.

#### Fakta penting tentang penyimpanan

Berbeda dengan beberapa database lain, PostgreSQL tidak mengoptimalkan storage berdasarkan perbedaan `CHAR`, `VARCHAR`, atau `TEXT`. Artinya, selama tidak ada pembatasan panjang, ketiganya disimpan dengan cara yang identik. Inilah alasan mengapa penggunaan `TEXT` di PostgreSQL sangat umum dan tidak dianggap “boros”.

Perbedaan utama justru terletak pada aturan validasi panjang data, bukan pada cara penyimpanannya.

#### Perbedaan penting untuk CHAR dan VARCHAR tanpa parameter

Bagian ini sering menjadi sumber kesalahan, terutama bagi pengguna yang terbiasa dengan database lain. Perhatikan contoh berikut:

```sql
-- PERHATIAN: Perilaku berbeda!
CREATE TABLE demo (
    col1 CHAR,       -- Sama dengan CHAR(1) - hanya 1 karakter!
    col2 VARCHAR,    -- Sama dengan TEXT - unlimited!
    col3 TEXT        -- Unlimited
);

INSERT INTO demo VALUES ('AB', 'AB', 'AB');
-- ERROR di kolom 'col1': value too long for type character(1)
-- OK untuk col2 dan col3
```

Pada definisi tabel di atas, `col1 CHAR` tanpa parameter secara default berarti `CHAR(1)`. Artinya, kolom ini hanya bisa menyimpan satu karakter. Ketika kita mencoba memasukkan nilai `'AB'`, PostgreSQL menolak data tersebut karena panjangnya melebihi satu karakter.

Sebaliknya, `VARCHAR` tanpa parameter di PostgreSQL diperlakukan sama seperti `TEXT`, yaitu tidak memiliki batas panjang. Oleh karena itu, nilai `'AB'` dapat disimpan dengan aman di `col2` dan `col3` tanpa error.

#### Kesimpulan praktis

Dari contoh ini, ada beberapa pelajaran penting yang bisa diambil:

- Jangan mengasumsikan bahwa `CHAR` tanpa parameter bersifat fleksibel — di PostgreSQL, itu berarti tepat satu karakter.
- `VARCHAR` tanpa parameter tidak sama dengan `VARCHAR(255)`; ia justru setara dengan `TEXT`.
- Pemilihan antara `VARCHAR(n)` dan `TEXT` sebaiknya didasarkan pada kebutuhan validasi panjang, bukan kekhawatiran soal penyimpanan.

Dengan memahami perbedaan perilaku ini, kita bisa menghindari error yang tidak perlu dan memilih tipe data karakter yang paling sesuai dengan kebutuhan aplikasi.

### b. CHAR(n) - Fixed-Length Character (❌ TIDAK DIREKOMENDASIKAN)

`CHAR(n)` adalah tipe data karakter dengan panjang tetap. Artinya, PostgreSQL selalu menyimpan nilai dengan panjang persis `n` karakter. Jika data yang dimasukkan lebih pendek dari panjang yang ditentukan, PostgreSQL akan otomatis menambahkan spasi di bagian akhir hingga mencapai panjang tersebut. Perilaku inilah yang menjadi sumber banyak masalah, sehingga tipe ini umumnya tidak direkomendasikan untuk penggunaan sehari-hari.

#### Cara kerja CHAR(n) dalam praktik

Perhatikan contoh berikut:

```sql
-- Membuat tabel dengan CHAR
CREATE TABLE example (
    abbreviation CHAR(5)
);

-- Insert data yang lebih pendek dari 5 karakter
INSERT INTO example VALUES ('A');

-- Hasil: 'A    ' (dipadding dengan 4 spasi)
SELECT * FROM example;
SELECT length(abbreviation) FROM example;  -- 5 (bukan 1!)
```

Ketika kita memasukkan nilai `'A'` ke dalam kolom `CHAR(5)`, PostgreSQL tidak menyimpannya sebagai satu karakter saja. Nilai tersebut dipadding dengan empat spasi di belakang, sehingga panjang totalnya menjadi 5 karakter. Hal ini bisa terlihat jelas ketika kita memanggil fungsi `length()`, yang mengembalikan nilai `5`, bukan `1`.

Sebaliknya, jika kita mencoba memasukkan data yang lebih panjang dari batas yang ditentukan, PostgreSQL akan langsung menolaknya:

```sql
-- Insert data yang lebih panjang dari 5 karakter
INSERT INTO example VALUES ('ABCDEF');  -- ERROR!
-- Error: value too long for type character(5)
```

Ini menunjukkan bahwa `CHAR(n)` hanya membatasi panjang maksimum, bukan memastikan data benar-benar memiliki panjang yang kita inginkan.

#### Masalah utama dengan CHAR(n)

Ada beberapa alasan kuat mengapa `CHAR(n)` dianggap bermasalah dan jarang direkomendasikan di PostgreSQL:

1. **Tidak menegakkan panjang minimum**
   Meskipun kolom didefinisikan sebagai `CHAR(5)`, kita tetap bisa memasukkan satu karakter saja. Jika tujuan kita adalah memastikan data selalu memiliki tepat 5 karakter, `CHAR(n)` tidak benar-benar membantu.

2. **Auto-padding dengan spasi**
   Spasi tambahan ini tidak selalu terlihat, tetapi tetap memakan ruang penyimpanan dan bisa menimbulkan kebingungan saat memproses data.

3. **Tidak ada keuntungan performa**
   Berbeda dengan anggapan umum, `CHAR(n)` tidak lebih cepat atau lebih efisien dibandingkan `VARCHAR` atau `TEXT` di PostgreSQL. Dari sisi penyimpanan dan akses, ketiganya diperlakukan hampir sama.

4. **Performance overhead tambahan**
   Auto-padding dan pengecekan panjang tetap membutuhkan kerja ekstra, baik dari sisi penggunaan storage maupun siklus CPU, meskipun efeknya sering kali tidak disadari.

5. **Perilaku yang tidak konsisten**
   Cara PostgreSQL memperlakukan spasi di akhir string bisa berbeda tergantung operator dan collation yang digunakan:

   - Operator `LIKE` menganggap spasi sebagai karakter yang signifikan.
   - Operator `=` pada kebanyakan collation mengabaikan spasi di bagian akhir.

Perbedaan perilaku ini bisa menjadi sumber bug yang sulit dilacak.

#### Contoh masalah nyata

Contoh berikut menggambarkan mengapa `CHAR(n)` sering menimbulkan hasil yang membingungkan:

```sql
CREATE TABLE codes (
    code CHAR(5)
);

INSERT INTO codes VALUES ('ABC');  -- Berhasil! (dipadding jadi 'ABC  ')
-- Tapi kita mau enforce HARUS 5 karakter!
```

Secara kasat mata, data `'ABC'` terlihat tidak memenuhi aturan 5 karakter. Namun, PostgreSQL tetap menerimanya dan menyimpannya sebagai `'ABC  '`.

Masalah menjadi lebih rumit saat melakukan perbandingan:

```sql
-- Perbandingan yang membingungkan (perilaku bisa berbeda tergantung collation):
SELECT 'ABC' = 'ABC  ';      -- Umumnya TRUE (trailing spaces diabaikan)
SELECT 'ABC' LIKE 'ABC  ';   -- FALSE (spasi dianggap!)
```

Dua ekspresi di atas memberikan hasil yang berbeda karena cara operator memperlakukan spasi tidak sama. Situasi seperti ini bisa menyebabkan bug halus yang sulit dideteksi, terutama dalam query yang kompleks.

Karena berbagai kelemahan tersebut, praktik yang umum direkomendasikan di PostgreSQL adalah menghindari `CHAR(n)` dan lebih memilih `VARCHAR(n)` atau `TEXT`, yang lebih konsisten, lebih fleksibel, dan lebih mudah dipahami perilakunya.

### c. VARCHAR(n) / CHARACTER VARYING(n) - Variable-Length dengan Limit

`VARCHAR(n)` atau `CHARACTER VARYING(n)` adalah tipe data karakter dengan panjang variabel, tetapi memiliki batas maksimum yang jelas. Berbeda dengan `CHAR(n)`, tipe ini tidak melakukan padding dengan spasi. PostgreSQL hanya menyimpan karakter yang benar-benar dimasukkan, sehingga lebih efisien dan perilakunya lebih intuitif.

#### Cara kerja VARCHAR(n)

Contoh berikut menunjukkan beberapa variasi penggunaan `VARCHAR`:

```sql
-- Dengan limit
CREATE TABLE users (
    last_name VARCHAR(255)
);

-- Tanpa limit (sama dengan TEXT)
CREATE TABLE posts (
    content VARCHAR  -- Tanpa angka = unlimited, identik dengan TEXT
);
```

Pada tabel `users`, kolom `last_name` dibatasi maksimal 255 karakter. PostgreSQL akan menolak data yang panjangnya melebihi batas tersebut.

Sementara itu, `VARCHAR` tanpa parameter angka diperlakukan sama seperti `TEXT`. Artinya, kolom `content` tidak memiliki batas panjang eksplisit dan dapat menyimpan teks sepanjang apa pun yang diperlukan.

#### Contoh insert dan hasilnya

```sql
-- Insert data
INSERT INTO users VALUES ('Smith');       -- OK, disimpan sebagai 'Smith' (5 chars)
INSERT INTO users VALUES ('A');           -- OK, disimpan sebagai 'A' (1 char)
INSERT INTO users VALUES ('Long name...(256 chars)');  -- ERROR jika > 255!
```

Dari contoh di atas, terlihat bahwa:

- Nilai `'Smith'` disimpan apa adanya sebagai 5 karakter.
- Nilai `'A'` tetap disimpan sebagai 1 karakter, tanpa tambahan spasi.
- Nilai yang panjangnya melebihi 255 karakter langsung ditolak oleh PostgreSQL.

Inilah keunggulan utama `VARCHAR(n)`: panjang data fleksibel, tetapi tetap ada pagar pembatas untuk mencegah data yang terlalu panjang.

#### Karakteristik utama VARCHAR(n)

Beberapa karakteristik penting dari `VARCHAR(n)` antara lain:

- Tidak ada padding dengan spasi di akhir data.
- Ukuran penyimpanan sesuai dengan panjang aktual data yang disimpan.
- Memungkinkan pembatasan panjang maksimal secara eksplisit.
- Pemilihan batas harus dilakukan dengan hati-hati, karena perubahan batas di kemudian hari bisa menjadi sulit, terutama jika data sudah banyak.

#### Peringatan penting saat memilih batas panjang

Menentukan nilai `n` yang terlalu kecil sering kali menjadi sumber masalah di masa depan. Contoh berikut menggambarkan risikonya:

```sql
-- HATI-HATI dengan batas yang terlalu kecil!
CREATE TABLE people (
    last_name VARCHAR(30)  -- Mungkin terlalu kecil!
);
```

Sekilas, batas 30 karakter terlihat cukup untuk nama belakang. Namun, dalam praktik, ada banyak nama yang sah dan valid dengan panjang lebih dari itu, misalnya:

- `"Wolfeschlegelsteinhausenbergerdorff"` (35 karakter)
- `"Hubert Blaine Wolfeschlegelsteinhausenbergerdorff"` (jauh lebih dari 30 karakter)

Nama-nama tersebut valid secara nyata, tetapi akan ditolak oleh database jika batasnya terlalu kecil.

Pendekatan yang lebih aman adalah:

```sql
-- Lebih aman:
CREATE TABLE people (
    last_name VARCHAR(255)  -- Atau gunakan TEXT dengan constraint
);
```

Dengan batas yang lebih longgar, atau bahkan menggunakan `TEXT` ditambah constraint terpisah, kita bisa menghindari penolakan data yang sebenarnya valid, sambil tetap menjaga kontrol atas kualitas dan konsistensi data.

### d. TEXT - Variable-Length Unlimited (✅ DIREKOMENDASIKAN)

`TEXT` adalah tipe data karakter dengan panjang variabel tanpa batas eksplisit. Di PostgreSQL, tipe ini direkomendasikan untuk sebagian besar kasus karena sederhana, fleksibel, dan tidak memiliki kekurangan performa dibandingkan `VARCHAR`. Secara perilaku dan implementasi, `TEXT` identik dengan `VARCHAR` tanpa parameter angka.

#### TEXT dan VARCHAR tanpa limit

Perhatikan contoh berikut:

```sql
-- TEXT dan VARCHAR tanpa limit adalah identik
CREATE TABLE articles (
    title TEXT,
    content VARCHAR  -- Sama persis dengan TEXT
);
```

Pada tabel di atas, kolom `title` bertipe `TEXT`, sedangkan `content` bertipe `VARCHAR` tanpa angka. Di PostgreSQL, kedua kolom ini diperlakukan dengan cara yang sama, baik dari sisi penyimpanan, performa, maupun perilaku saat query dijalankan.

Tipe ini juga mampu menyimpan data dengan ukuran yang sangat besar:

```sql
-- Bisa menyimpan data yang sangat besar (hingga ~1GB)
INSERT INTO articles VALUES (
    'My Article',
    'Konten yang sangat panjang... (bisa sampai ~1GB!)'
);
```

PostgreSQL mendukung penyimpanan string hingga sekitar 1 GB per nilai, sehingga `TEXT` cocok untuk konten artikel, deskripsi panjang, atau data teks lain yang ukurannya sulit diprediksi sejak awal.

#### Keuntungan menggunakan TEXT

Ada beberapa alasan kuat mengapa `TEXT` sering menjadi pilihan terbaik:

- Tidak memiliki batas panjang eksplisit, dengan batas maksimum sekitar 1 GB.
- Sangat fleksibel untuk data yang panjangnya tidak dapat diprediksi.
- Performa dan efisiensi penyimpanan sama dengan `VARCHAR`.
- Tetap memungkinkan penambahan constraint di kemudian hari jika ternyata dibutuhkan pembatasan panjang atau aturan lain.

Dengan kata lain, memilih `TEXT` tidak berarti kehilangan kontrol. Kita hanya menunda keputusan pembatasan panjang sampai benar-benar diperlukan.

#### Catatan penting tentang kesetaraan TEXT dan VARCHAR

Di PostgreSQL, kesetaraan antara `TEXT` dan `VARCHAR` tanpa parameter bukan sekadar konsep, tetapi benar-benar diterapkan secara internal.

```sql
-- Kedua kolom ini 100% identik
CREATE TABLE demo (
    col1 TEXT,
    col2 VARCHAR
);
```

Kedua kolom di atas tidak memiliki perbedaan apa pun dari sudut pandang PostgreSQL. Tidak ada perbedaan performa, cara penyimpanan, maupun kemampuan query.

#### Batasan maksimal penyimpanan

Meskipun disebut “unlimited”, `TEXT` tetap memiliki batas maksimum teknis. Panjang string yang dapat disimpan dibatasi sekitar **1 GB** per nilai. Nilai maksimum yang diizinkan untuk parameter `n` pada tipe lain seperti `VARCHAR(n)` juga harus berada di bawah batas ini.

Dalam praktik sehari-hari, batas tersebut sangat jarang tercapai, sehingga `TEXT` dapat dianggap aman dan bebas batas untuk hampir semua kebutuhan aplikasi.

### e. TOAST - The Oversized-Attribute Storage Technique

TOAST adalah mekanisme internal PostgreSQL yang dirancang khusus untuk menangani data berukuran sangat besar, terutama untuk tipe seperti `TEXT`, `BYTEA`, atau tipe lain yang panjangnya bisa membengkak. Semua ini bekerja secara otomatis dan transparan, sehingga sebagai pengguna atau developer, kita tidak perlu melakukan konfigurasi khusus atau mengubah cara menulis query.

#### Cara kerja TOAST

Secara garis besar, TOAST bekerja ketika ukuran sebuah nilai teks melewati ambang tertentu, biasanya sekitar 2 KB.

Alurnya kurang lebih sebagai berikut:

1. Ketika sebuah kolom teks menjadi cukup besar (sekitar ≥ 2 KB), PostgreSQL mendeteksinya secara otomatis.
2. Data besar tersebut tidak disimpan langsung di baris utama tabel.
3. PostgreSQL memindahkan data ke tabel terpisah yang disebut TOAST table, yang tidak terlihat dan tidak bisa diakses langsung oleh user.
4. Baris utama di tabel asli tetap kecil dan ringkas, hanya menyimpan referensi ke data TOAST.
5. Saat data besar itu benar-benar dibutuhkan, PostgreSQL akan mengambilnya dari TOAST table secara otomatis.

Semua proses ini terjadi di balik layar dan sepenuhnya transparan bagi aplikasi.

#### Contoh penggunaan dalam praktik

Contoh berikut menunjukkan bagaimana TOAST bekerja tanpa kita sadari:

```sql
-- Contoh: Menyimpan artikel yang sangat panjang
CREATE TABLE blog_posts (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT  -- Jika besar, auto-TOAST!
);

-- Insert artikel 1MB
INSERT INTO blog_posts (title, content)
VALUES ('Long Article', repeat('Lorem ipsum... ', 100000));
```

Pada contoh ini, kolom `content` berisi teks yang sangat panjang, sekitar 1 MB. PostgreSQL akan secara otomatis:

- Menyimpan kolom kecil seperti `id` dan `title` langsung di tabel utama, sehingga tetap ringkas.
- Memindahkan isi `content` ke TOAST table karena ukurannya besar.
- Menyediakan mekanisme untuk mengambil kembali data tersebut saat kita melakukan `SELECT`.

Semua ini terjadi tanpa perubahan apa pun pada cara kita melakukan `INSERT` atau `SELECT`.

#### Analogi sederhana

Untuk mempermudah pemahaman, TOAST bisa dianalogikan seperti ini:

- Tabel utama adalah sebuah rumah kecil.
- TOAST table adalah gudang penyimpanan di belakang rumah.
- Barang-barang kecil disimpan di rumah.
- Barang besar otomatis dipindahkan ke gudang.
- Kita tetap bisa mengambil barang dari gudang kapan pun dibutuhkan, tanpa repot memindahkannya sendiri.

Analogi ini menggambarkan bagaimana PostgreSQL menjaga agar “rumah” tetap rapi dan efisien, tanpa kehilangan akses ke data besar.

#### Keuntungan menggunakan TOAST

Keberadaan TOAST memberikan beberapa keuntungan penting:

- Bekerja sepenuhnya otomatis, tanpa perlu konfigurasi tambahan.
- Menjaga ukuran baris tetap kecil dan efisien, sehingga operasi scan tabel menjadi lebih cepat.
- Memberikan performa yang lebih baik untuk query yang tidak membutuhkan kolom besar, karena PostgreSQL tidak perlu selalu membaca data TOAST.
- Konsep serupa juga ada di database lain seperti MySQL, meskipun dengan nama dan implementasi yang berbeda.

Dengan TOAST, PostgreSQL mampu menangani data teks berukuran besar secara elegan, tanpa membebani struktur tabel utama dan tanpa menyulitkan developer dalam penggunaannya.

### f. Pertimbangan untuk Index

Saat bekerja dengan index di PostgreSQL, ada satu batasan teknis yang sangat penting untuk dipahami: ukuran maksimum satu baris index adalah **2712 bytes**. Batas ini berlaku per entri index, bukan per kolom tabel. Karena itu, penggunaan kolom bertipe `TEXT` pada index perlu dipertimbangkan dengan hati-hati, terutama jika data yang disimpan berpotensi sangat panjang.

#### Masalah batas ukuran index

Perhatikan contoh berikut:

```sql
-- Masalah: Index limit 2712 bytes
CREATE TABLE urls (
    long_url TEXT
);
CREATE INDEX idx_url ON urls(long_url);
-- Bisa error jika data > 2712 bytes!
-- ERROR: index row size exceeds maximum
```

Secara sintaks, definisi tabel dan index di atas terlihat valid. Namun, ketika data `long_url` melebihi batas ukuran index (2712 bytes), PostgreSQL bisa menolak operasi insert atau update dengan error karena ukuran baris index terlalu besar. TOAST tidak membantu di sini, karena data yang disimpan di index tidak di-TOAST dengan cara yang sama seperti data di tabel.

#### Solusi untuk mengatasi batas index

Ada beberapa pendekatan yang umum digunakan untuk menghindari masalah ini, tergantung kebutuhan aplikasi.

##### Solusi 1: Batasi panjang dengan VARCHAR

Pendekatan paling sederhana adalah menggunakan `VARCHAR` dengan batas panjang yang aman untuk kolom yang akan di-index.

```sql
CREATE TABLE urls (
    long_url VARCHAR(500)  -- Aman untuk di-index
);
CREATE INDEX idx_url ON urls(long_url);
```

Dengan membatasi panjang data sejak awal, kita memastikan bahwa ukuran entri index tidak akan melampaui batas yang ditentukan PostgreSQL.

##### Solusi 2: TEXT dengan CHECK constraint

Jika ingin tetap menggunakan `TEXT`, kita bisa menambahkan constraint untuk membatasi panjang data secara eksplisit.

```sql
CREATE TABLE urls (
    long_url TEXT CHECK (length(long_url) <= 500)
);
CREATE INDEX idx_url ON urls(long_url);
```

Pendekatan ini memberikan fleksibilitas tipe `TEXT`, sekaligus perlindungan agar data yang di-index tidak terlalu panjang.

##### Solusi 3: Expression index

Alternatif lain adalah membuat index berdasarkan ekspresi, misalnya hanya pada sebagian awal string.

```sql
CREATE TABLE urls (
    long_url TEXT  -- Tetap unlimited
);
CREATE INDEX idx_url ON urls(substring(long_url, 1, 255));
```

Dengan cara ini, hanya 255 karakter pertama yang masuk ke index. Ini sering kali cukup untuk kebutuhan pencarian atau perbandingan, tanpa harus mengindeks seluruh isi kolom.

##### Solusi 4: Conditional partial index

Pendekatan yang lebih selektif adalah menggunakan partial index dengan kondisi tertentu.

```sql
CREATE INDEX idx_url ON urls(long_url)
WHERE length(long_url) <= 255;
```

Index ini hanya akan mencakup baris-baris dengan panjang `long_url` yang masih aman untuk di-index. Baris dengan data yang lebih panjang tidak dimasukkan ke index sama sekali.

#### Kapan batas index ini menjadi masalah

Batasan ukuran index biasanya mulai terasa pada kasus-kasus berikut:

- URL yang sangat panjang.
- Nilai hash atau token yang panjang.
- Data hasil serialisasi seperti JSON atau XML yang disimpan sebagai `TEXT`.
- Konten buatan pengguna yang kemudian di-index.

Dalam skenario seperti ini, penggunaan `TEXT` tanpa pembatasan pada kolom yang di-index hampir pasti akan menimbulkan masalah.

#### Best practice untuk kolom yang di-index

Praktik terbaik yang umum diterapkan adalah memisahkan kolom yang perlu di-index dan kolom yang tidak.

```sql
-- Untuk kolom yang akan di-index: lebih aman pakai VARCHAR dengan limit
CREATE TABLE articles (
    title VARCHAR(255),  -- Di-index, batasi dengan aman
    slug VARCHAR(255),   -- Di-index, batasi dengan aman
    content TEXT,        -- Tidak di-index, bisa unlimited
    summary TEXT         -- Tidak di-index, bisa unlimited
);

CREATE INDEX idx_title ON articles(title);
CREATE INDEX idx_slug ON articles(slug);
-- Tidak perlu index untuk content dan summary
```

Dengan pendekatan ini, kolom yang sering digunakan untuk pencarian atau lookup memiliki batas panjang yang aman dan mudah di-index, sementara kolom yang berisi teks panjang tetap fleksibel tanpa dibatasi.

## 3. Hubungan Antar Konsep

### Hirarki Rekomendasi (dari terbaik ke terburuk):

```
1. TEXT (dengan check constraint jika perlu)
   ↓
2. VARCHAR(n) (dengan n yang cukup besar, terutama untuk indexed columns)
   ↓
3. VARCHAR (tanpa limit, sama dengan TEXT)
   ↓
4. CHAR(n) ❌ JANGAN DIGUNAKAN!
```

### Proses Pengambilan Keputusan:

```
Apakah kolom akan di-index?
├─ Ya → Gunakan VARCHAR(n) dengan n yang reasonable (< 2712 bytes)
│        Atau TEXT + CHECK constraint
└─ Tidak
    └─ Apakah panjang HARUS tetap (misal: kode negara 'ID', 'US')?
        ├─ Ya → Gunakan TEXT/VARCHAR + CHECK constraint (bukan CHAR!)
        └─ Tidak
            └─ Apakah ada batas maksimal yang masuk akal?
                ├─ Ya, dan pasti tidak akan berubah
                │   → VARCHAR(n) dengan n yang cukup besar
                │   (Contoh: judul blog post max 75 char - kita yang kontrol)
                └─ Tidak, atau batasnya tidak pasti
                    → TEXT
                    (Contoh: nama orang - identitas mereka, kita tidak bisa batasi)
```

### Perbandingan dengan Database Lain:

| Database   | CHAR(n)          | VARCHAR(n)      | TEXT          |
| ---------- | ---------------- | --------------- | ------------- |
| PostgreSQL | ❌ Hindari       | ✅ OK           | ✅✅ Terbaik  |
| MySQL      | Bisa lebih cepat | Standard choice | Tipe terpisah |
| SQL Server | Bisa lebih cepat | Standard choice | Tipe terpisah |

**PostgreSQL berbeda!** - Semua tipe char disimpan sama, tidak seperti database lain.

## 4. Kesimpulan

PostgreSQL memiliki tiga tipe data karakter yang **sebenarnya identik** dalam hal penyimpanan dan performa: CHAR(n), VARCHAR(n), dan TEXT. Ini berbeda dengan database lain di mana CHAR mungkin lebih cepat.

**Rekomendasi utama:**

1. **Jangan gunakan CHAR(n)** - Tidak ada keuntungan, banyak kerugian
2. **Gunakan TEXT sebagai default untuk kolom tanpa index** - Fleksibel, performa bagus
3. **Gunakan VARCHAR(n) untuk kolom yang di-index** - Hindari index size limit (2712 bytes)
4. **Gunakan VARCHAR(n) hanya jika:** Anda tahu pasti batasnya dan itu masuk akal
5. **Untuk data identitas orang:** Selalu beri ruang yang cukup (TEXT atau VARCHAR(255))
6. **Untuk data yang Anda kontrol:** VARCHAR(n) dengan batas yang reasonable
7. **Ingat:** CHAR tanpa parameter = CHAR(1), VARCHAR tanpa parameter = TEXT

PostgreSQL punya TOAST yang otomatis menangani text yang sangat besar, jadi tidak perlu khawatir tentang performa untuk kolom TEXT yang besar **selama kolom tersebut tidak di-index**.

**Decision Tree Sederhana:**

```
Kolom akan di-index?
├─ Ya → VARCHAR(255) atau TEXT + CHECK(length <= 255)
└─ Tidak → TEXT (default), atau VARCHAR(n) jika perlu enforcement
```

**Key takeaway:** Di PostgreSQL, TEXT adalah pilihan paling aman dan fleksibel untuk hampir semua kasus **kecuali untuk kolom yang di-index**. Untuk kolom yang di-index, gunakan VARCHAR(n) dengan batas yang reasonable atau TEXT dengan CHECK constraint untuk menghindari index size limitation. Tambahkan CHECK constraint untuk enforcement yang proper, bukan mengandalkan CHAR(n) yang bermasalah.
