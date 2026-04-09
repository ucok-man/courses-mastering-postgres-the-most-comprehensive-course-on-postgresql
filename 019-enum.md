# 📝 Tipe Data Enum di PostgreSQL (VERSI TERKOREKSI)

## 1. Ringkasan Singkat

Video ini membahas tentang tipe data Enum di PostgreSQL, yang merupakan tipe data unik yang menggabungkan **human readability** (terlihat seperti string untuk manusia) dengan **fixed-size storage** (disimpan dengan ukuran tetap 4 bytes di database). Enum sangat cocok untuk data dengan pilihan tetap dan terbatas seperti status, mood, atau ukuran. Video juga membahas cara menambah/menghapus nilai enum, urutan sorting yang unik, serta perbandingan dengan alternatif lain seperti check constraints.

## 2. Konsep Utama

### a. Karakteristik Dasar Enum

Enum di PostgreSQL memiliki karakteristik yang cukup unik karena ia menjembatani kebutuhan manusia dan kebutuhan sistem database. Untuk memahaminya dengan benar, kita perlu melihat Enum dari dua sudut pandang sekaligus: bagaimana ia terlihat oleh developer, dan bagaimana ia disimpan oleh PostgreSQL secara internal.

#### Dual Nature dari Enum

Enum memiliki dua “wajah” yang berbeda tergantung siapa yang berinteraksi dengannya.

Dari sisi manusia atau developer, nilai Enum terlihat dan digunakan seperti string biasa. Kita menulis `'happy'`, `'sad'`, atau `'neutral'`, membacanya dengan jelas, dan langsung memahami maknanya tanpa perlu melihat dokumentasi tambahan.

Namun dari sisi database, Enum sama sekali bukan string. PostgreSQL menyimpannya sebagai data berukuran tetap (fixed size) sebesar **4 bytes**. Nilai ini bukan integer umum yang bisa kita manipulasi secara bebas, melainkan representasi internal yang mengacu pada **OID (Object Identifier)** dari baris yang tersimpan di system catalog `pg_enum`.

Penting untuk dicatat bahwa meskipun OID diimplementasikan sebagai unsigned four-byte integer, angka tersebut **bukan** menunjukkan urutan ordinal Enum (misalnya nilai pertama, kedua, dan seterusnya). Artinya, kita tidak boleh mengasumsikan bahwa nilai Enum bisa dibandingkan atau diurutkan berdasarkan angka internalnya tanpa aturan PostgreSQL.

#### Realita Penyimpanan (Storage Reality)

Sering muncul anggapan bahwa Enum selalu lebih hemat ruang dibandingkan text. Kenyataannya sedikit lebih nuansa dari itu.

Secara kasar, perbandingan penyimpanan antara Enum dan text adalah sebagai berikut:

- Enum: menggunakan **4 bytes**, ditambah kemungkinan alignment padding
- Text: menggunakan **1-4 byte overhead**, ditambah panjang string sebenarnya

Jika kita bandingkan secara konkret:

- `'CA'`
  Disimpan sebagai text: 1 byte overhead + 2 karakter = **3 bytes**
  Disimpan sebagai enum: **4 bytes**
  Dalam kasus ini, **text justru lebih kecil**

- `'draft'`
  Text: 1 + 5 = **6 bytes**
  Enum: **4 bytes**
  Di sini, **enum lebih efisien**

- `'published'`
  Text: 1 + 9 = **10 bytes**
  Enum: **4 bytes**
  Enum jauh lebih hemat

Dari perbandingan ini, kita bisa menarik kesimpulan penting: Enum lebih efisien secara storage untuk string dengan panjang **4 karakter atau lebih**, tetapi bisa menjadi kurang efisien jika nilai yang disimpan sangat pendek.

#### Definisi dan Sifat Dasar Enum

Secara konsep, Enum adalah **finite fixed list**, yaitu daftar nilai yang jumlahnya terbatas dan ditentukan di awal. Anda harus mendeklarasikan semua nilai yang diperbolehkan sejak awal, dan database tidak akan menerima nilai di luar daftar tersebut.

Inilah kekuatan utama Enum: ia memberikan **validasi data otomatis** langsung di level data. Tanpa perlu menambahkan `CHECK constraint` secara manual, PostgreSQL sudah memastikan bahwa hanya nilai yang valid yang bisa masuk ke kolom tersebut.

Pendekatan ini sangat cocok untuk data dengan domain yang jelas dan jarang berubah, seperti status, tipe, kategori tetap, atau state dalam workflow.

#### Contoh Deklarasi dan Penggunaan Enum

Berikut contoh sederhana bagaimana Enum didefinisikan dan digunakan di PostgreSQL.

Pertama, kita membuat tipe Enum baru:

```sql
CREATE TYPE mood AS ENUM (
    'happy',
    'sad',
    'neutral'
);
```

Dengan perintah ini, PostgreSQL mencatat tipe `mood` beserta seluruh nilai validnya di system catalog.

Selanjutnya, kita gunakan Enum tersebut pada sebuah tabel:

```sql
CREATE TABLE enum_example (
    current_mood mood
);
```

Kolom `current_mood` sekarang hanya boleh berisi salah satu dari nilai Enum yang sudah didefinisikan.

Ketika kita melakukan insert data:

```sql
INSERT INTO enum_example (current_mood) VALUES
    ('happy'),
    ('sad'),
    ('neutral');
```

Semua baris di atas akan berhasil karena nilainya sesuai dengan daftar Enum. Jika kita mencoba memasukkan nilai lain, misalnya `'angry'`, PostgreSQL akan langsung menolak dengan error. Inilah contoh nyata bagaimana Enum membantu menjaga konsistensi dan validitas data secara otomatis di level database.

### b. Kapan Menggunakan dan Tidak Menggunakan Enum

Setelah memahami karakteristik dan cara kerja Enum, langkah berikutnya adalah mengetahui **kapan Enum menjadi pilihan yang tepat** dan **kapan justru sebaiknya dihindari**. Keputusan ini penting karena Enum bersifat kaku (rigid) dibandingkan tipe data lain, sehingga salah memilih bisa berdampak pada fleksibilitas sistem di masa depan.

#### Kapan Sebaiknya Menggunakan Enum

Enum sangat cocok digunakan ketika domain data sudah jelas, stabil, dan tidak banyak berubah seiring waktu.

##### Pilihan Tetap dan Jarang Berubah

Gunakan Enum jika daftar nilainya sudah ditentukan sejak awal dan kecil kemungkinan berubah karena perubahan bisnis.

Contoh klasiknya adalah status sebuah blog post. Status seperti `draft`, `published`, atau `archived` biasanya bersifat permanen dan jarang sekali mengalami penambahan atau penghapusan nilai.

```sql
CREATE TYPE post_status AS ENUM (
    'draft',
    'published',
    'archived',
    'destroyed'
);
```

Dengan pendekatan ini, database secara otomatis memastikan bahwa kolom status hanya bisa berisi nilai yang valid. Tidak ada risiko typo atau status “liar” yang tidak terdefinisi.

##### Pilihan Terbatas dan Finite dengan Nilai String Panjang

Enum juga masuk akal jika nilai-nilainya berupa string yang relatif panjang dan jumlah opsinya terbatas. Dalam kondisi ini, Enum lebih efisien secara storage dibandingkan `TEXT`.

Contohnya ukuran pakaian dengan penamaan deskriptif:

```sql
CREATE TYPE shirt_size AS ENUM (
    'extra_small',
    'small',
    'medium',
    'large',
    'extra_large'
);
```

Setiap nilai di atas akan disimpan sebagai referensi 4 bytes, sementara jika menggunakan text, panjang string yang cukup besar akan memakan ruang lebih banyak.

##### Membutuhkan Urutan (Sort Order) yang Bermakna

Enum memungkinkan kita menentukan urutan nilai secara eksplisit berdasarkan urutan deklarasinya. Ini berguna ketika urutan logis tidak sama dengan urutan alfabet.

Sebagai contoh, urutan status workflow seperti `draft → published → archived` memiliki makna bisnis yang jelas. Dengan Enum, PostgreSQL dapat melakukan sorting berdasarkan urutan ini tanpa perlu logika tambahan.

#### Kapan Sebaiknya Tidak Menggunakan Enum

Di sisi lain, ada beberapa kondisi di mana penggunaan Enum justru akan menyulitkan pengembangan dan maintenance sistem.

##### Pilihan Sering Berubah

Jika bisnis Anda sering menambah, menghapus, atau memodifikasi nilai, Enum bukan pilihan yang tepat.

Alasannya, perubahan pada Enum di PostgreSQL tidak semudah update data biasa. Menambahkan nilai memang relatif mudah, tetapi **menghapus atau mengubah nilai Enum lama cukup rumit**, sering kali memerlukan drop dan recreate tipe Enum tersebut, yang berisiko pada data dan dependensi.

Dalam kasus seperti ini, tabel referensi atau lookup table jauh lebih fleksibel.

##### Membutuhkan Lookup Table dengan Data Tambahan

Enum hanya menyimpan nilai itu sendiri, tanpa kemampuan menyimpan metadata tambahan. Jika setiap pilihan membutuhkan informasi ekstra, Enum akan menjadi batasan.

Contoh yang sering terjadi adalah data negara bagian atau wilayah. Menggunakan Enum untuk ini adalah desain yang buruk:

```sql
CREATE TYPE us_state AS ENUM ('CA', 'NY', 'TX', ...);
```

Pendekatan yang lebih baik adalah menggunakan tabel terpisah:

```sql
CREATE TABLE states (
    code VARCHAR(2) PRIMARY KEY,
    name VARCHAR(100),
    capital VARCHAR(100),
    population INTEGER
);
```

Dengan tabel terpisah, Anda bisa menyimpan atribut tambahan, melakukan join, dan memperbarui data tanpa mengubah struktur tipe data.

##### Nilai String Sangat Pendek (≤ 3 Karakter)

Untuk string yang sangat pendek, Enum justru kalah efisien secara storage.

Contoh yang kurang ideal:

```sql
CREATE TYPE state_code AS ENUM ('CA', 'NY', 'TX');  -- 4 bytes each
```

Setiap nilai Enum tetap memakan 4 bytes, sementara jika menggunakan text atau varchar dengan validasi, ukurannya bisa lebih kecil:

```sql
state_code TEXT
CHECK (state_code IN ('CA', 'NY', 'TX'))
```

Dalam kasus ini, text hanya membutuhkan 1 byte overhead ditambah 2 karakter, sehingga totalnya lebih hemat dibandingkan Enum.

Kesimpulannya, Enum adalah alat yang sangat kuat jika digunakan pada konteks yang tepat. Gunakan Enum untuk domain yang stabil, terbatas, dan jelas. Namun, untuk data yang dinamis, membutuhkan metadata tambahan, atau sangat kecil ukurannya, pendekatan lain akan jauh lebih fleksibel dan efisien.

### c. Sort Order: Fitur Unik Enum

Salah satu fitur Enum yang sering terlewat, tetapi sangat powerful, adalah **cara PostgreSQL melakukan sorting terhadap nilai Enum**. Perilaku ini berbeda secara fundamental dibandingkan kolom bertipe `TEXT` atau `VARCHAR`, dan jika dipahami dengan benar, bisa sangat membantu dalam desain data.

#### Perbedaan dengan Kolom Text

Pada kolom bertipe text, PostgreSQL melakukan sorting secara **alfabetis (lexicographical order)**. Artinya, urutan hasil ditentukan berdasarkan perbandingan karakter, bukan makna bisnis dari nilai tersebut.

Enum bekerja dengan cara yang berbeda. Nilai Enum **tidak diurutkan secara alfabetis**, melainkan berdasarkan **urutan saat Enum dideklarasikan**. Urutan ini bersifat eksplisit dan melekat pada tipe Enum itu sendiri.

Perhatikan contoh berikut:

```sql
-- Deklarasi enum
CREATE TYPE mood AS ENUM ('happy', 'sad', 'neutral');

-- Insert data
INSERT INTO enum_example (current_mood) VALUES
    ('neutral'), ('happy'), ('sad'), ('happy');
```

Pada saat data dimasukkan, urutannya acak. Namun ketika kita menjalankan query dengan `ORDER BY`:

```sql
SELECT * FROM enum_example ORDER BY current_mood;
```

Hasilnya akan mengikuti urutan deklarasi Enum, yaitu:

```
happy
happy
sad
neutral
```

Bukan urutan alfabetis seperti:

```
happy
happy
neutral
sad
```

Hal ini terjadi karena PostgreSQL membandingkan nilai Enum berdasarkan posisi internalnya, yang sesuai dengan urutan saat Enum didefinisikan.

#### Keuntungan dari Sort Order Enum

Keunggulan utama dari mekanisme sorting ini adalah kita bisa mendefinisikan **urutan yang bermakna secara bisnis**, bukan sekadar urutan huruf.

Contoh yang sangat umum adalah ukuran pakaian. Secara alfabetis, urutannya justru tidak masuk akal, tetapi secara logis kita sudah tahu urutan yang benar.

```sql
CREATE TYPE size AS ENUM (
    'extra_small',  -- 1
    'small',        -- 2
    'medium',       -- 3
    'large',        -- 4
    'extra_large'   -- 5
);
```

Dengan deklarasi seperti ini, ketika kolom bertipe `size` diurutkan menggunakan `ORDER BY`, hasilnya akan mengikuti urutan:

```
extra_small → small → medium → large → extra_large
```

Atau dalam istilah yang lebih familiar:

```
XS → S → M → L → XL
```

Tanpa Enum, jika kita menggunakan text, hasil sorting akan menjadi:

```
extra_large → extra_small → large → medium → small
```

yang jelas tidak sesuai dengan logika domain.

Kesimpulannya, sort order bawaan Enum adalah fitur unik yang menjadikannya sangat cocok untuk data yang memiliki **urutan alami atau urutan proses**. Dengan satu deklarasi di awal, kita mendapatkan sorting yang konsisten, sederhana, dan bebas dari logika tambahan di query.

### d. Menambah Nilai ke Enum yang Sudah Ada

Salah satu keterbatasan Enum adalah sifatnya yang relatif kaku, namun PostgreSQL tetap menyediakan cara aman untuk **menambahkan nilai baru** ke Enum yang sudah ada. Penting untuk dipahami bahwa perubahan ini bukan sekadar menambah string baru, tetapi juga memengaruhi **urutan internal Enum**, yang nantinya berdampak pada hasil sorting.

#### Menambah Nilai di Posisi Terakhir

Cara paling sederhana adalah menambahkan nilai baru di akhir urutan Enum. Jika kita tidak menentukan posisi tertentu, PostgreSQL akan otomatis menempatkan nilai baru sebagai elemen terakhir.

```sql
ALTER TYPE mood ADD VALUE 'excited';
```

Dengan perintah ini, nilai `'excited'` ditambahkan setelah semua nilai yang sudah ada. Jika sebelumnya urutannya adalah:

```
happy → sad → neutral
```

maka setelah perintah di atas, urutannya menjadi:

```
happy → sad → neutral → excited
```

Pendekatan ini aman dan umum digunakan jika nilai baru memang secara logis berada di akhir workflow atau kategori.

#### Menambah Nilai Sebelum Nilai Tertentu (BEFORE)

Jika nilai baru harus berada di posisi tertentu, PostgreSQL menyediakan opsi `BEFORE`. Ini berguna ketika urutan Enum mencerminkan tahapan atau prioritas tertentu.

```sql
ALTER TYPE mood ADD VALUE 'afraid' BEFORE 'sad';
```

Perintah ini akan menyisipkan `'afraid'` tepat sebelum `'sad'`. Urutan Enum sekarang menjadi:

```
happy → afraid → sad → neutral → excited
```

Dengan cara ini, kita bisa mempertahankan urutan yang bermakna tanpa harus mendefinisikan ulang seluruh Enum.

#### Menambah Nilai Setelah Nilai Tertentu (AFTER)

Selain `BEFORE`, kita juga bisa menambahkan nilai baru tepat setelah nilai tertentu menggunakan `AFTER`.

```sql
ALTER TYPE mood ADD VALUE 'melancholic' AFTER 'afraid';
```

Hasilnya, `'melancholic'` ditempatkan setelah `'afraid'` dan sebelum `'sad'`, sehingga urutan lengkapnya menjadi:

```
happy → afraid → melancholic → sad → neutral → excited
```

Opsi ini sangat berguna ketika Enum digunakan untuk sorting dan urutan tersebut memiliki makna bisnis yang jelas.

#### Catatan Penting Terkait Idempotency

Sejak PostgreSQL versi 9.3, kita bisa menggunakan `IF NOT EXISTS` saat menambahkan nilai Enum. Ini sangat penting dalam konteks migrasi database, agar script bisa dijalankan berulang kali tanpa menyebabkan error.

```sql
ALTER TYPE mood ADD VALUE IF NOT EXISTS 'excited';
```

Dengan opsi ini, PostgreSQL hanya akan menambahkan `'excited'` jika nilai tersebut belum ada. Jika sudah ada, perintah akan dilewati tanpa error.

#### Visualisasi Perubahan Urutan

Untuk membantu memahami dampak setiap operasi, berikut gambaran evolusi urutan Enum:

```
Awal:       happy → sad → neutral
Add end:    happy → sad → neutral → excited
Add before: happy → afraid → sad → neutral → excited
Add after:  happy → afraid → melancholic → sad → neutral → excited
```

Dari sini terlihat jelas bahwa setiap penambahan nilai ke Enum bukan hanya soal menambah opsi baru, tetapi juga soal **menjaga urutan yang konsisten dan bermakna**. Oleh karena itu, desain Enum sebaiknya dipikirkan sejak awal, terutama jika urutan nilainya akan digunakan dalam operasi `ORDER BY`.

### e. Menghapus/Mengubah Nilai Enum (SANGAT KOMPLEKS!)

Menghapus atau mengubah nilai Enum adalah salah satu operasi paling berisiko di PostgreSQL. Berbeda dengan menambah nilai, operasi ini **tidak didukung secara langsung** dan membutuhkan pendekatan khusus agar database tetap aman dan konsisten.

#### Peringatan Kritis

PostgreSQL **tidak menyediakan perintah resmi** untuk menghapus nilai Enum yang sudah ada. Meskipun secara teknis nilai Enum disimpan di system catalog `pg_enum`, **menghapus baris secara manual dari tabel tersebut sangat berbahaya** dan tidak boleh dilakukan.

Risikonya nyata dan serius:

- Nilai Enum yang “dihapus” masih bisa tersisa di halaman index tingkat atas (upper index pages)
- Index dapat menjadi tidak konsisten
- Dalam skenario terburuk, database bisa mengalami **corruption** yang sulit dipulihkan

Karena itu, pendekatan “hack” atau manipulasi langsung system catalog harus dihindari sepenuhnya.

#### Solusi Aman: Drop dan Recreate dengan Pendekatan Transaksional

Satu-satunya cara yang aman untuk “menghapus” atau mengganti nilai Enum adalah dengan **membuat tipe Enum baru**, memigrasikan data, lalu mengganti tipe lama. Proses ini memang terlihat panjang, tetapi inilah harga yang harus dibayar demi integritas data.

##### Langkah 1: Membuat Enum Baru

Pertama, buat tipe Enum baru yang hanya berisi nilai-nilai yang memang ingin dipertahankan.

```sql
CREATE TYPE mood_new AS ENUM ('happy', 'sad', 'neutral', 'afraid');
```

Enum `mood_new` ini adalah versi “bersih” dari Enum lama, tanpa nilai yang ingin dihapus atau diubah.

##### Langkah 2: Migrasi Data di Dalam Transaction

Seluruh proses migrasi sebaiknya dibungkus dalam sebuah transaction. Ini penting agar perubahan bersifat atomik.

```sql
BEGIN;
```

Sebelum mengubah tipe kolom, pastikan tidak ada data yang menggunakan nilai Enum lama yang sudah tidak valid. Jika ada, kita harus menormalkannya terlebih dahulu.

```sql
UPDATE enum_example
SET current_mood = 'neutral'
WHERE current_mood NOT IN ('happy', 'sad', 'neutral', 'afraid');
```

Query ini memastikan bahwa semua baris memiliki nilai yang kompatibel dengan Enum baru.

Setelah data aman, barulah kita ubah tipe kolom ke Enum baru:

```sql
ALTER TABLE enum_example
ALTER COLUMN current_mood TYPE mood_new
USING current_mood::TEXT::mood_new;
```

Bagian `USING` sangat penting. PostgreSQL perlu tahu bagaimana cara mengonversi nilai lama ke tipe baru. Di sini, nilainya dikonversi ke `TEXT` terlebih dahulu, lalu dipetakan ke `mood_new`.

Jika semua langkah berhasil, kita bisa menyelesaikan transaction:

```sql
COMMIT;
```

##### Langkah 3: Membersihkan Enum Lama (Opsional)

Setelah kolom sudah sepenuhnya menggunakan Enum baru, Enum lama tidak lagi dibutuhkan.

```sql
DROP TYPE mood;
ALTER TYPE mood_new RENAME TO mood;
```

Dengan cara ini, nama tipe tetap konsisten dari sudut pandang aplikasi, tetapi isinya sudah diperbarui.

#### Mengapa Transaction Sangat Penting

Penggunaan transaction di sini bukan sekadar praktik yang baik, tetapi **keharusan**.

Transaction memastikan bahwa:

- Semua perubahan terjadi secara **atomik** (berhasil semua atau gagal semua)
- Tidak ada data “setengah jadi” jika proses terhenti di tengah jalan
- Jika terjadi error pada salah satu langkah, seluruh perubahan otomatis di-rollback ke kondisi semula

Intinya, setiap kali Anda perlu menghapus atau mengubah nilai Enum, anggaplah itu sebagai operasi migrasi skema yang serius. Rencanakan dengan matang, gunakan transaction, dan jangan pernah memodifikasi system catalog secara langsung.

### f. Melihat "Under the Hood" Enum

Untuk benar-benar memahami bagaimana Enum bekerja di PostgreSQL, kita perlu melihat apa yang terjadi di balik layar. Pada level permukaan, Enum terlihat seperti string biasa, tetapi secara internal mekanismenya jauh lebih kompleks dan dirancang untuk efisiensi serta konsistensi.

#### Melihat Representasi Internal Enum

PostgreSQL menyimpan metadata Enum di system catalog bernama `pg_enum`. Kita bisa melihat seluruh definisi Enum yang ada di database dengan query berikut:

```sql
SELECT * FROM pg_catalog.pg_enum;
```

Hasil query ini akan menampilkan setiap nilai Enum beserta informasi internal yang digunakan PostgreSQL untuk mengelolanya.

#### Koreksi Penting tentang Cara Enum Disimpan

Ada kesalahpahaman umum bahwa Enum disimpan sebagai integer 1, 2, 3, dan seterusnya. Ini **tidak benar**.

Secara internal:

- Setiap nilai Enum direpresentasikan sebagai **OID (Object Identifier)**
- OID adalah identifier unik berukuran **4 bytes**
- OID ini **bukan** ordinal position dari Enum

Walaupun OID diimplementasikan sebagai unsigned four-byte integer, angka tersebut tidak memiliki hubungan langsung dengan urutan Enum yang kita definisikan. Artinya, kita tidak boleh mengandalkan nilai numerik OID untuk logika bisnis atau sorting.

#### Memahami Kolom Penting di `pg_enum`

Ketika kita melihat isi `pg_enum`, ada beberapa kolom yang penting untuk dipahami:

- `enumtypid`
  Menunjukkan ID tipe Enum yang bersangkutan. Semua nilai Enum dengan `enumtypid` yang sama berasal dari satu tipe Enum yang sama.

- `enumlabel`
  Inilah label Enum yang kita kenal dan gunakan di query, seperti `'happy'`, `'sad'`, dan seterusnya.

- `enumsortorder`
  Ini adalah kolom yang menentukan **urutan sorting Enum**. Nilainya berupa numeric atau float, dan **inilah yang digunakan PostgreSQL saat melakukan `ORDER BY`**, bukan OID.

Penting dicatat bahwa `enumsortorder` bukan representasi internal nilai Enum itu sendiri, melainkan metadata khusus untuk keperluan pengurutan.

#### Bagaimana PostgreSQL Menangani INSERT BEFORE dan AFTER

Ketika kita menambahkan nilai Enum menggunakan `BEFORE` atau `AFTER`, PostgreSQL tidak menggeser ulang semua nilai Enum yang sudah ada. Sebaliknya, ia menggunakan teknik yang dikenal sebagai **“split the difference”** pada kolom `enumsortorder`.

Ilustrasinya kira-kira seperti ini:

```
Awal:     happy(1.0) → sad(2.0) → neutral(3.0)
+ before: happy(1.0) → afraid(1.5) → sad(2.0) → neutral(3.0)
+ after:  happy(1.0) → afraid(1.5) → melancholic(1.75) → sad(2.0) → neutral(3.0)
```

PostgreSQL menghitung nilai di tengah-tengah dua `enumsortorder` yang berdekatan, lalu menyimpannya sebagai urutan baru. Pendekatan ini memungkinkan penambahan nilai Enum di posisi mana pun tanpa harus melakukan rewrite besar-besaran terhadap metadata yang sudah ada.

Sekali lagi, perlu ditekankan bahwa teknik ini berlaku pada **`enumsortorder`**, bukan pada OID atau representasi internal nilai Enum.

#### Fungsi Utility untuk Bekerja dengan Enum

PostgreSQL juga menyediakan beberapa fungsi bawaan yang sangat membantu saat bekerja dengan Enum.

Untuk melihat semua nilai Enum dalam urutan yang benar:

```sql
SELECT enum_range(NULL::mood);
```

Hasilnya adalah array berisi seluruh nilai Enum sesuai urutan deklarasi dan modifikasi:

```
{happy,afraid,melancholic,sad,neutral,excited}
```

Kita juga bisa mengambil subset nilai Enum dalam sebuah rentang tertentu:

```sql
SELECT enum_range('afraid'::mood, 'sad'::mood);
```

Hasilnya:

```
{afraid,melancholic,sad}
```

Selain itu, tersedia juga fungsi untuk mengambil nilai pertama dan terakhir dari sebuah Enum:

```sql
SELECT enum_first(NULL::mood);  -- Nilai pertama
SELECT enum_last(NULL::mood);   -- Nilai terakhir
```

Fungsi-fungsi ini sangat berguna untuk validasi, pembuatan UI (misalnya dropdown), atau logika aplikasi yang perlu memahami urutan Enum tanpa hardcode nilai-nilainya.

### g. Alternatif: Check Constraints vs Enum

Enum bukan satu-satunya cara untuk membatasi nilai sebuah kolom. PostgreSQL menyediakan beberapa pendekatan alternatif yang masing-masing memiliki trade-off berbeda antara fleksibilitas, keterbacaan, dan efisiensi storage. Memahami perbandingan ini akan membantu Anda memilih solusi yang paling tepat sesuai kebutuhan domain data.

#### Text dengan Check Constraint

Pendekatan paling sederhana adalah menggunakan kolom `TEXT` lalu menambahkan `CHECK constraint` untuk membatasi nilai yang diperbolehkan.

```sql
CREATE TABLE posts (
    status TEXT CHECK (status IN ('draft', 'published', 'archived'))
);
```

Dengan definisi ini, kolom `status` tetap bertipe text, tetapi database akan menolak nilai apa pun di luar daftar yang sudah ditentukan.

Keuntungan utama pendekatan ini adalah kemudahan maintenance. Jika suatu saat bisnis membutuhkan status baru, Anda hanya perlu melakukan `ALTER TABLE` untuk memperbarui constraint, tanpa harus membuat atau menghapus tipe data.

Keuntungannya antara lain:

- Lebih mudah di-maintain karena perubahan cukup dilakukan pada constraint
- Tidak perlu `CREATE TYPE` atau `DROP TYPE`
- Sangat fleksibel terhadap perubahan kebutuhan
- Lebih efisien secara storage untuk string yang sangat pendek (kurang dari 4 karakter)

Namun, ada juga keterbatasannya:

- Untuk string yang panjang (lebih dari 4 karakter), storage akan lebih besar dibandingkan Enum
- Tidak memiliki sort order khusus; pengurutan akan selalu alfabetis

Pendekatan ini cocok untuk domain data yang masih mungkin berkembang dan belum sepenuhnya stabil.

#### Domain dengan Check Constraint

Jika Anda menyukai fleksibilitas `CHECK constraint` tetapi ingin validasi yang bisa digunakan ulang, PostgreSQL menyediakan fitur **DOMAIN**.

```sql
CREATE DOMAIN post_status AS TEXT
CHECK (VALUE IN ('draft', 'published', 'archived'));

CREATE TABLE posts (
    status post_status
);
```

Domain pada dasarnya adalah tipe data kustom yang dibangun di atas tipe dasar (dalam hal ini `TEXT`), lengkap dengan aturan validasinya.

Keuntungan pendekatan ini:

- Constraint bersifat reusable dan bisa dipakai di banyak tabel
- Perubahan aturan cukup dilakukan di satu tempat
- Validasi menjadi terpusat dan konsisten

Domain sangat berguna ketika banyak tabel membutuhkan aturan validasi yang sama, tetapi Anda tetap menginginkan fleksibilitas seperti pada `TEXT + CHECK constraint`.

#### Plain Integer

Pendekatan lain yang sering digunakan adalah menyimpan status sebagai angka integer.

```sql
CREATE TABLE posts (
    status INTEGER CHECK (status BETWEEN 1 AND 4)
);
-- 1=draft, 2=published, 3=archived, 4=destroyed
```

Dari sisi storage, pendekatan ini sangat efisien. Integer disimpan dalam 4 bytes, sama seperti Enum, dan mudah diubah jika rentangnya masih masuk dalam constraint.

Keuntungannya:

- Storage compact (4 bytes per nilai)
- Relatif mudah di-maintain dari sisi skema

Namun, kekurangannya cukup signifikan:

- Nilai bersifat **opaque** dan tidak human-readable
- Tanpa dokumentasi atau mapping di aplikasi, angka seperti `1`, `2`, atau `3` tidak memiliki makna
- Berpotensi membingungkan rekan satu tim dan meningkatkan risiko bug

Pendekatan ini hanya masuk akal jika seluruh makna nilai dikelola dengan ketat di application layer.

#### Perbandingan Storage dan Karakteristik

Jika dibandingkan secara ringkas, masing-masing pendekatan memiliki karakteristik berikut:

- **Enum**
  Menggunakan storage tetap 4 bytes per nilai, sangat readable, tetapi sulit diubah. Cocok untuk string dengan panjang ≥ 4 karakter dan domain yang stabil.

- **Text + Check Constraint**
  Storage bergantung pada panjang string, sangat fleksibel, dan mudah diubah. Cocok untuk berbagai panjang nilai dan kebutuhan yang sering berubah.

- **Integer**
  Storage tetap 4 bytes, tetapi hampir tidak readable tanpa konteks tambahan. Cocok jika logika sepenuhnya dikendalikan di application layer.

- **Domain**
  Storage sama dengan tipe dasarnya, sangat fleksibel, dan cocok untuk validasi yang ingin dipakai ulang di banyak tabel.

#### Contoh Kasus Nyata

Untuk status dengan label yang relatif panjang, Enum biasanya lebih efisien:

```sql
CREATE TYPE long_status AS ENUM ('draft', 'published', 'archived', 'destroyed');
```

Setiap nilai disimpan dalam 4 bytes, terlepas dari panjang stringnya.

Sebaliknya, untuk kode yang sangat pendek seperti kode negara bagian, `TEXT + CHECK constraint` justru lebih hemat:

```sql
CREATE TABLE states (
    code TEXT CHECK (code IN ('CA', 'NY', 'TX'))
);
```

Setiap nilai hanya membutuhkan 1 byte overhead ditambah 2 karakter, sehingga lebih kecil dibandingkan Enum.

Kesimpulannya, tidak ada satu pendekatan yang selalu benar. Enum unggul dalam konsistensi dan sort order, sementara `CHECK constraint`, Domain, dan Integer menawarkan fleksibilitas yang lebih tinggi. Pilihan terbaik selalu bergantung pada stabilitas domain data, kebutuhan perubahan, dan prioritas antara keterbacaan, performa, dan maintenance.

## 3. Hubungan Antar Konsep

**Alur pemahaman Enum:**

```
1. Deklarasi Type
   ↓
2. Nilai disimpan sebagai OID (4 bytes) internally
   ↓
3. OID merujuk ke pg_enum catalog row
   ↓
4. Ditampilkan sebagai string untuk human readability
   ↓
5. Sort order berdasarkan enumsortorder field (bukan OID, bukan alfabetis!)
   ↓
6. Validasi otomatis saat insert/update
   ↓
7. Dapat ditambah nilai baru (di akhir/before/after)
   ↓
8. TIDAK BISA hapus nilai secara aman (bisa corrupt index)
```

**Trade-offs antara pilihan (UPDATED):**

```
Fixed values + Long strings (≥4 chars) + Meaningful order → Enum ⭐
Fixed values + Short strings (<4 chars) → Text + Check ⭐
Frequently changing values → Text + Check Constraint ⭐
Need additional data per value → Lookup Table ⭐
Reusable across tables → Domain ⭐
Application-only usage → Integer (acceptable)
```

## 4. Kesimpulan

Enum di PostgreSQL adalah **tipe data hybrid** yang menggabungkan:

- ✅ **Fixed-size storage** (4 bytes OID)
- ✅ **Human readability** (ditampilkan sebagai string)
- ✅ **Data validation** (hanya nilai yang dideklarasikan)
- ✅ **Meaningful sort order** (berdasarkan enumsortorder, bukan alfabetis)

**Kapan menggunakan:**

- Pilihan **tetap dan terbatas** dengan string ≥ 4 karakter
- Perlu **readable representation** di database
- Jarang memerlukan **perubahan nilai**
- Meaningful order penting

**Trade-offs:**

- ⚠️ **TIDAK BISA** remove values secara aman
- ⚠️ Sort order **tidak alfabetis** (bisa jadi gotcha)
- ⚠️ Kurang fleksibel dibanding text + check constraint
- ⚠️ Tidak selalu lebih compact (4 bytes fixed vs text variable)

**Alternatif yang viable:**

- **Text + Check Constraint**: Lebih fleksibel, lebih baik untuk string pendek
- **Domain**: Reusable, mudah maintain
- **Lookup Table**: Paling fleksibel, untuk complex cases
- **Plain Integer**: Compact tapi tidak readable

**Prinsip akhir:**

- Pilih enum jika list fixed, string panjang (≥4 chars), dan meaningful order penting
- Untuk frequent changes atau string pendek, gunakan check constraints
- Untuk complex requirements, gunakan lookup table
- Selalu pertimbangkan storage efficiency berdasarkan panjang string actual
