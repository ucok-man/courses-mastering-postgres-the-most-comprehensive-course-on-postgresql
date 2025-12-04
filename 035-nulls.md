# 📝 Nulls dan Constraint NOT NULL di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang constraint NOT NULL di PostgreSQL, dengan penekanan pada karakteristik unik dari nilai NULL dalam database dan best practices dalam penggunaannya. Materi ini merupakan bagian akhir dari modul tentang tipe data dan constraint dasar.

## 2. Konsep Utama

### a. Karakteristik Khusus NULL

#### Memahami sifat unik dari NULL

Dalam SQL, `NULL` adalah nilai yang sering membingungkan karena memiliki karakteristik yang berbeda dari nilai biasa. Banyak orang mengira bahwa `NULL` adalah “kosong” atau “nol”, padahal itu tidak benar. `NULL` mewakili sesuatu yang _tidak diketahui_, sehingga database tidak bisa memperlakukannya seperti angka, string, atau boolean.

Untuk memperjelas, berikut beberapa sifat pentingnya:

- `NULL` **bukan angka nol** (`0`)
- `NULL` **bukan false**
- `NULL` **bukan string kosong** (`""`)
- `NULL` **bukan nilai kosong** seperti array atau objek tanpa isi
- `NULL` benar-benar berarti: _nilai ini tidak diketahui atau tidak ada informasinya_

Dengan kata lain, **NULL menyatakan “unknown value”**, bukan “nothing” dalam arti teknis.

#### Bagaimana perbandingan terhadap NULL bekerja

Salah satu hal yang sering mengejutkan adalah hasil operasi perbandingan dengan `NULL`. Misalnya:

```sql
SELECT 1 = NULL;  -- Hasilnya bukan TRUE atau FALSE, tapi NULL (unknown)
```

Ketika SQL diminta membandingkan sebuah nilai yang jelas (`1`) dengan sesuatu yang tidak diketahui (`NULL`), database tidak bisa mengambil keputusan apakah keduanya sama atau tidak. Karena tidak ada informasi tentang nilai sebenarnya di sisi kanan, hasilnya adalah `NULL`.

Analogi sederhananya seperti ini:
Bayangkan Anda memegang sebuah angka di tangan, tetapi saya tidak boleh melihatnya. Jika saya bertanya, “Apakah angka itu sama dengan 1?”, Anda tidak bisa menjawab _ya_ atau _tidak_, karena saya tidak tahu apa yang Anda sembunyikan. Satu-satunya jawaban yang mungkin adalah:
“Saya tidak tahu.”
Itulah yang terjadi ketika SQL melakukan perbandingan dengan `NULL`.

#### Cara yang benar untuk memeriksa NULL

Kesalahan umum pemula adalah menggunakan operator perbandingan seperti `=` atau `!=` saat mencari baris yang memiliki nilai `NULL`. Ini tidak akan pernah bekerja karena perbandingan langsung dengan `NULL` selalu menghasilkan nilai `unknown`.

Contoh kesalahan umum:

```sql
-- ❌ SALAH: Tidak akan mengembalikan baris apa pun
SELECT * FROM products WHERE name = NULL;
```

Cara yang benar adalah menggunakan operator khusus:

```sql
-- ✅ BENAR: Menggunakan IS NULL
SELECT * FROM products WHERE name IS NULL;
```

Operator `IS NULL` dan `IS NOT NULL` dibuat secara khusus untuk menangani nilai yang tidak diketahui. Dengan begitu, Anda bisa mencari data yang benar-benar tidak memiliki nilai (unknown) dengan hasil yang konsisten.

Penggunaan `IS NULL` ini sangat penting ketika bekerja dengan data yang tidak lengkap atau ketika kolom tertentu memang boleh tidak memiliki nilai.

### b. Three-Valued Boolean Logic

#### Konsep dasar logika tiga nilai

Dalam SQL, khususnya ketika bekerja dengan `NULL`, kita tidak hanya berhadapan dengan dua kemungkinan nilai boolean seperti pada pemrograman biasa (TRUE dan FALSE). SQL menggunakan **Three-Valued Logic (3VL)**, yaitu sistem logika yang memiliki tiga hasil:

1. **TRUE**
2. **FALSE**
3. **NULL (unknown)**

Nilai ketiga inilah yang membuat perilaku `NULL` terasa berbeda. Ketika `NULL` ikut masuk ke dalam sebuah ekspresi logika atau perbandingan, SQL sering kali tidak bisa memastikan apakah hasilnya benar atau salah, sehingga hasilnya menjadi: **unknown**.

#### Bagaimana perbandingan menjadi “unknown”

Mari lihat contoh konkret:

```sql
SELECT 'Aaron' = NULL;  -- Hasil: NULL (bukan TRUE atau FALSE)
```

Kenapa hasilnya bukan FALSE saja?
Karena SQL tidak tahu apa nilai `NULL` sebenarnya. Bisa saja nilai yang hilang itu memang `'Aaron'`, atau `'Bob'`, atau sesuatu yang lain. Karena tidak ada informasi, SQL tidak dapat menyimpulkan. Akhirnya, hasil yang diberikan adalah **NULL**, yaitu “saya tidak tahu”.

#### NULL dibandingkan dengan NULL

Hal yang sering mengejutkan adalah bahwa `NULL` tidak dianggap sama dengan `NULL`.

```sql
SELECT NULL = NULL;  -- Hasil: NULL
```

Ini terjadi karena SQL memandang `NULL` sebagai “nilai yang tidak diketahui”. Membandingkan dua hal yang sama-sama tidak diketahui tetap tidak memberikan informasi apa pun. Ibaratnya:
“Kita punya dua kotak tertutup. Apakah isi keduanya sama?”
Selama tidak dibuka, jawabannya tetap: “Tidak tahu.”

#### Implikasi penggunaan Three-Valued Logic

Pemahaman tentang 3VL sangat penting saat menulis query yang melibatkan kondisi, terutama di bagian `WHERE`, `JOIN`, atau filter kompleks. Jika tidak berhati-hati, baris yang memiliki `NULL` sering kali tidak ikut terpilih karena hasil kondisinya menjadi “unknown” dan dianggap tidak memenuhi filter.

Inilah mengapa SQL menyediakan operator khusus seperti `IS NULL` dan `IS NOT NULL`—agar Anda bisa menangani kasus “unknown” secara eksplisit, bukan mengandalkan perbandingan yang menghasilkan hasil ambigu.

### c. Constraint NOT NULL

#### Cara kerja dan tujuan penggunaan NOT NULL

Constraint `NOT NULL` digunakan untuk memastikan bahwa sebuah kolom _selalu memiliki nilai_ ketika baris baru disimpan. Dengan kata lain, kolom tersebut tidak boleh dibiarkan kosong atau tidak diisi. Ini merupakan salah satu bentuk **data integrity constraint** yang paling dasar dan paling sering digunakan dalam desain database.

Pada dasarnya, `NOT NULL` memberi tahu database:
“Kolom ini wajib diisi. Tidak boleh ada nilai yang tidak diketahui (NULL).”

#### Perbandingan dua cara mendefinisikan kolom yang wajib memiliki nilai

Kadang ada orang yang mencoba menerapkan aturan “kolom tidak boleh NULL” menggunakan `CHECK constraint`. Secara teknis, ini bisa dilakukan, tetapi bukan cara yang tepat.

Contoh yang kurang tepat:

```sql
-- ❌ Cara yang tidak direkomendasikan (menggunakan CHECK constraint)
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT CHECK (name IS NOT NULL)  -- Tidak efisien
);
```

Secara fungsi, ini memang mencegah nilai NULL. Namun, cara ini **tidak efisien** dan membuat schema menjadi kurang jelas. Selain itu, database sudah memiliki mekanisme khusus untuk menangani hal ini, sehingga menggunakan `CHECK` justru membingungkan dan membuat performa lebih buruk.

Cara yang benar adalah menggunakan constraint khususnya:

```sql
-- ✅ Cara yang direkomendasikan
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,  -- Lebih efisien dan jelas
    price NUMERIC NOT NULL
);
```

Di sini, schema lebih mudah dibaca: kita bisa langsung melihat mana kolom yang wajib diisi tanpa harus menelusuri ekspresi `CHECK`.

#### Menggabungkan NOT NULL dengan logika domain menggunakan CHECK

`NOT NULL` sering digunakan bersamaan dengan `CHECK` untuk menerapkan aturan bisnis yang lebih spesifik. Contoh:

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC NOT NULL CHECK (price > 0)  -- Menggabungkan NOT NULL dan domain logic
);
```

Penjelasannya:

- `name TEXT NOT NULL` memastikan setiap produk pasti memiliki nama.
- `price NUMERIC NOT NULL` memastikan harga harus diberikan.
- `CHECK (price > 0)` menegakkan aturan bisnis bahwa harga harus bernilai positif.

Constraint ini bekerja bersama-sama. Meskipun `price` sudah diberi `NOT NULL`, kita tetap membutuhkan `CHECK (price > 0)` karena dalam bisnis tertentu, harga `0` dianggap tidak valid atau sama saja dengan tidak memberikan nilai sama sekali.

Dengan kombinasi tersebut, database menjadi lebih kuat dalam menjaga kualitas data, dan kesalahan input dapat dicegah sejak awal sebelum data masuk ke tabel.

### d. Primary Key dan NOT NULL

#### Hubungan otomatis antara PRIMARY KEY dan NOT NULL

Saat Anda mendeklarasikan sebuah kolom sebagai **PRIMARY KEY**, SQL secara otomatis menambahkan dua aturan penting pada kolom tersebut:

1. **Kolom tidak boleh NULL (NOT NULL)**
2. **Nilai dalam kolom tersebut harus unik (UNIQUE)**

Akibatnya, Anda tidak perlu lagi menuliskan `NOT NULL` secara terpisah. Database sudah menganggap bahwa sebuah primary key _wajib memiliki nilai_ dan _tidak boleh duplikat_.

Perhatikan contoh berikut:

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
```

Dalam definisi tabel ini:

- Kolom `id` sudah otomatis menjadi **NOT NULL**, meskipun tidak dituliskan secara eksplisit.
- Kolom tersebut juga memiliki constraint **UNIQUE**, karena setiap baris harus memiliki `id` yang berbeda.
- Penggunaan `GENERATED ALWAYS AS IDENTITY` memungkinkan database menghasilkan nilai incremental otomatis untuk setiap baris baru.

Dengan demikian, menambah `NOT NULL` secara manual seperti ini tidak diperlukan:

```sql
-- ❌ Tidak perlu melakukan ini
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY NOT NULL
```

Karena efeknya sama saja, dan penulisan tersebut hanya membuat schema lebih panjang tanpa alasan.

Secara keseluruhan, PRIMARY KEY sudah mencakup semua yang diperlukan untuk sebuah identifier unik: nilai harus ada, dan tidak boleh sama dengan baris lainnya.

### e. Perilaku Default (Nullable)

#### Kolom secara default bersifat nullable

Dalam SQL, setiap kolom yang Anda buat **secara otomatis dianggap boleh berisi NULL** kecuali Anda secara eksplisit memberikan constraint `NOT NULL`. Ini adalah aturan dasar yang sering terlewat oleh banyak orang ketika baru belajar merancang schema database.

Artinya, jika Anda tidak menuliskan apa pun pada kolom tersebut, SQL akan menganggap bahwa kolom itu _boleh tidak diisi_, dan nilai default-nya bisa berupa `NULL`.

Perhatikan contoh berikut:

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT,        -- Nullable (default)
    description TEXT  -- Nullable (default)
);
```

Pada definisi tabel di atas:

- Kolom `name` tidak diberi `NOT NULL`, sehingga sifatnya **nullable**.
- Kolom `description` juga **nullable** secara default.
- Satu-satunya kolom yang pasti tidak boleh NULL adalah `id`, karena kolom tersebut adalah primary key.

#### Contoh perilaku penyimpanan data

Karena kolom `name` dan `description` bersifat nullable, SQL tidak memaksa Anda untuk selalu memberikan nilai pada kolom tersebut ketika melakukan `INSERT`.

Contohnya:

```sql
-- Ini valid:
INSERT INTO products (name) VALUES (NULL);  -- ✅ Berhasil
```

Perintah di atas berhasil disimpan karena:

- `name` boleh berisi NULL.
- `description` juga nullable dan tidak wajib diisi.
- Kolom `id` akan diisi otomatis oleh mekanisme `GENERATED ALWAYS AS IDENTITY`.

Ini menunjukkan bahwa suatu kolom akan menerima nilai `NULL` selama Anda tidak memberikan constraint yang memaksanya untuk selalu memiliki nilai. Memahami perilaku default ini penting agar Anda dapat merancang schema yang konsisten dengan kebutuhan aplikasi dan aturan bisnis.

## 3. Hubungan Antar Konsep

### Alur Pemahaman:

1. **NULL sebagai fondasi**: Memahami bahwa NULL adalah "unknown value" yang berperilaku berbeda dari nilai normal
2. **Implikasi three-valued logic**: Karena NULL adalah unknown, operasi perbandingan menghasilkan NULL (bukan TRUE/FALSE)
3. **Kebutuhan akan NOT NULL**: Untuk menghindari kompleksitas NULL, gunakan constraint NOT NULL
4. **Manfaat NOT NULL**: Dengan membatasi NULL, operasi seperti:

   - Indexing
   - Comparing
   - Grouping
   - Sorting
   - Querying

   Menjadi lebih sederhana dan efisien

### Best Practice Decision Tree:

```
Apakah kolom ini secara logika HARUS memiliki nilai?
├─ YA → Gunakan NOT NULL
└─ TIDAK → Biarkan nullable (default)
    └─ JANGAN membuat "fake null" (seperti menggunakan string kosong untuk merepresentasikan NULL)
```

## 4. Kesimpulan

- **NULL adalah "unknown value"**, bukan nilai kosong biasa (bukan 0, false, atau empty string)
- Perbandingan dengan NULL menghasilkan **three-valued logic** (TRUE, FALSE, atau NULL)
- **Best practice**: Jadikan NOT NULL sebagai default untuk kolom Anda, kecuali ada alasan kuat untuk membiarkannya nullable
- **Sintaks yang benar**: `column_name TYPE NOT NULL` (bukan menggunakan CHECK constraint)
- **Manfaat NOT NULL**: Enforcement nilai wajib ada + query operations yang lebih efisien
- Constraint NOT NULL bisa dikombinasikan dengan CHECK constraint untuk domain logic tambahan
- PRIMARY KEY secara otomatis adalah NOT NULL

**Prinsip emas**: "Make every column NOT NULL unless you have a very good reason to make it nullable."
