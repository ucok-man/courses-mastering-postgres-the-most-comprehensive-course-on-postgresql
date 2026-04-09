# 📝 Arrays di PostgreSQL

## 1. Ringkasan Singkat

Materi ini membahas tentang **arrays** sebagai tipe data standalone di PostgreSQL (berbeda dengan arrays di dalam JSON). PostgreSQL memiliki dukungan yang sangat robust untuk arrays dengan banyak fungsi dan operator untuk manipulasi data. Video ini menjelaskan cara mendeklarasikan array columns, kapan sebaiknya menggunakan arrays versus tabel terpisah, cara insert data array, dan preview beberapa operator powerful untuk bekerja dengan arrays seperti indexing, slicing, filtering, dan `unnest`.

## 2. Konsep Utama

### a. Apa itu Array Data Type di PostgreSQL?

#### Pengertian Dasar

Di PostgreSQL, **Array** adalah salah satu tipe data yang berdiri sendiri (standalone). Artinya, array ini bukan bagian dari format lain seperti JSON, melainkan benar-benar tipe kolom tersendiri di dalam database.

Sederhananya, array memungkinkan kita menyimpan **lebih dari satu nilai dalam satu kolom**, selama semua nilai tersebut memiliki tipe data yang sama.

Contoh sederhana:

```sql
CREATE TABLE mahasiswa (
    nama TEXT,
    nilai INTEGER[]
);
```

Pada contoh di atas:

- Kolom `nama` menyimpan teks biasa
- Kolom `nilai` adalah array dari integer (`INTEGER[]`)

Kita bisa memasukkan data seperti ini:

```sql
INSERT INTO mahasiswa (nama, nilai)
VALUES ('Andi', ARRAY[80, 85, 90]);
```

Artinya, satu baris data bisa menyimpan banyak nilai sekaligus dalam kolom `nilai`.

#### Karakteristik Array di PostgreSQL

Ada beberapa hal penting yang perlu dipahami tentang array:

- **Setiap tipe data bisa dijadikan array**
  Misalnya:
  - `TEXT[]` → array berisi teks
  - `INTEGER[]` → array berisi angka
  - `DATE[]` → array berisi tanggal

- **Typed (harus satu jenis data)**
  Semua elemen dalam array harus memiliki tipe yang sama. Tidak bisa mencampur integer dengan string dalam satu array.

- **Didukung banyak fungsi dan operator**
  PostgreSQL menyediakan berbagai fungsi bawaan untuk mengolah array, seperti:
  - Mengambil elemen tertentu
  - Mengecek apakah nilai ada dalam array
  - Menggabungkan array
  - dan lain-lain

Contoh akses elemen:

```sql
SELECT nilai[1] FROM mahasiswa;
```

Perlu diperhatikan, hasilnya adalah elemen pertama dari array.

#### Perbedaan dengan JSON Array

Meskipun terlihat mirip, array di PostgreSQL berbeda dengan array di dalam JSON. Berikut penjelasan yang lebih mudah dipahami:

- **JSON Array**
  - Bagian dari dokumen JSON
  - Index dimulai dari **0** (zero-based)
  - Bisa menyimpan berbagai tipe data dalam satu array (fleksibel)

- **Native Array PostgreSQL**
  - Tipe kolom langsung di database
  - Index dimulai dari **1** (one-based) ⚠️
  - Harus satu tipe data (typed)

Contoh perbedaan indexing:

```sql
-- PostgreSQL Array
SELECT ARRAY[10, 20, 30][1];  -- hasil: 10

-- JSON Array
SELECT '[10, 20, 30]'::json->0;  -- hasil: 10
```

Di sini terlihat jelas:

- Array PostgreSQL dimulai dari index **1**
- JSON array dimulai dari index **0**

#### Kapan Menggunakan Array?

Array sangat berguna ketika:

- Data memiliki jumlah item yang tidak tetap
- Data tersebut masih satu kategori/tipe
- Tidak ingin membuat tabel relasi tambahan

Namun, tetap perlu bijak dalam penggunaannya. Untuk data yang kompleks atau relasional, biasanya lebih baik menggunakan tabel terpisah daripada array.

Kesimpulannya, array di PostgreSQL adalah fitur yang **powerful dan fleksibel**, tetapi paling efektif digunakan pada kasus tertentu yang memang membutuhkan penyimpanan multi-value dalam satu kolom.

### b. Cara Mendeklarasikan Array Columns

#### Konsep Dasar Deklarasi Array

Di PostgreSQL, kita bisa membuat kolom yang menyimpan array dengan beberapa cara. Intinya, kita hanya perlu memberi tahu bahwa sebuah kolom tidak berisi satu nilai saja, tetapi **sekumpulan nilai dengan tipe yang sama**.

PostgreSQL menyediakan **dua sintaks berbeda**, tetapi hasilnya sama persis (equivalent).

#### Sintaks 1: Menggunakan Square Brackets `[]`

Ini adalah cara yang paling umum dan paling sering digunakan karena jelas dan eksplisit.

```sql
CREATE TABLE array_example (
    id BIGINT PRIMARY KEY,
    integer_array INTEGER[],      -- Array of integers
    text_array TEXT[],            -- Array of text
    boolean_array BOOLEAN[]       -- Array of booleans
);
```

Penjelasannya:

- `INTEGER[]` → berarti kolom ini berisi array angka
- `TEXT[]` → array string
- `BOOLEAN[]` → array true/false

Dengan pendekatan ini, tipe data langsung terlihat jelas dari deklarasinya.

Contoh penggunaan:

```sql
INSERT INTO array_example (id, integer_array)
VALUES (1, ARRAY[1, 2, 3, 4]);
```

#### Sintaks 2: Menggunakan Keyword `ARRAY`

Alternatif lain adalah menggunakan keyword `ARRAY`:

```sql
CREATE TABLE array_example (
    id BIGINT PRIMARY KEY,
    integer_array ARRAY,
    text_array ARRAY,
    boolean_array ARRAY
);
```

Secara konsep, ini dianggap setara dengan:

- `integer_array ARRAY` ≈ `INTEGER[]`
- `text_array ARRAY` ≈ `TEXT[]`
- `boolean_array ARRAY` ≈ `BOOLEAN[]`

Namun dalam praktik nyata, pendekatan ini **lebih jarang digunakan**, karena:

- Tidak eksplisit menunjukkan tipe elemen
- Bisa membingungkan saat membaca struktur tabel

Jadi, meskipun valid, biasanya developer lebih memilih sintaks `[]`.

#### Kesimpulan Perbandingan Sintaks

- Keduanya **valid dan equivalent**
- `[]` lebih:
  - jelas
  - eksplisit
  - mudah dibaca

- `ARRAY` lebih:
  - singkat
  - tetapi kurang informatif

#### Nested Arrays (Multidimensional)

PostgreSQL juga mendukung **array bertingkat** (multidimensional array), yaitu array di dalam array.

Contoh deklarasi:

```sql
CREATE TABLE array_example (
    id BIGINT PRIMARY KEY,
    nested_array INTEGER[][]      -- Array 2 dimensi
);
```

Artinya:

- Ini bukan sekadar daftar angka
- Tapi **array yang berisi array angka**

Contoh isi datanya:

```sql
INSERT INTO array_example (id, nested_array)
VALUES (1, ARRAY[[1,2,3], [4,5,6]]);
```

Strukturnya bisa dibayangkan seperti tabel:

```
[
  [1, 2, 3],
  [4, 5, 6]
]
```

Kita juga bisa menuliskannya dengan gaya `ARRAY`:

```sql
nested_array ARRAY ARRAY
```

Namun lagi-lagi, pendekatan ini kurang umum karena tidak sejelas `INTEGER[][]`.

#### Kapan Multidimensional Array Digunakan?

Biasanya digunakan ketika:

- Data memiliki struktur grid atau matriks
- Contohnya:
  - data koordinat
  - matrix perhitungan
  - representasi tabel kecil dalam satu kolom

#### Kesimpulan

Mendeklarasikan array di PostgreSQL sebenarnya cukup sederhana:

- Gunakan `[]` untuk cara yang paling jelas dan umum
- Gunakan multidimensional array (`[][]`) jika memang butuh struktur bertingkat
- Hindari penggunaan `ARRAY` tanpa tipe jika ingin kode lebih mudah dipahami

Dengan memahami cara deklarasi ini, kamu sudah siap untuk mulai menggunakan array secara efektif di dalam desain database.

### c. Cara Insert Data ke Array Columns

#### Konsep Dasar Insert Array

Di PostgreSQL, kita bisa memasukkan data ke dalam kolom array dengan beberapa cara. Secara umum, PostgreSQL menyediakan **dua format utama**:

1. Format yang lebih eksplisit dan mudah dibaca
2. Format yang lebih ringkas dan compact

Keduanya valid, dan bisa digunakan sesuai kebutuhan atau preferensi.

#### Format 1: Constructor Syntax (Lebih Jelas)

Format ini menggunakan keyword `ARRAY[...]`, sehingga terlihat sangat jelas bahwa kita sedang membuat array.

```sql
INSERT INTO array_example (integer_array, text_array, boolean_array)
VALUES (
    ARRAY[1, 2, 3],                    -- Integer array
    ARRAY['rose', 'daisy', 'poppy'],   -- Text array
    ARRAY[true, false, true]           -- Boolean array
);
```

Penjelasan:

- `ARRAY[1, 2, 3]` → array berisi angka
- `ARRAY['rose', 'daisy', 'poppy']` → array berisi teks
- `ARRAY[true, false, true]` → array boolean

Kelebihan format ini:

- Lebih **jelas dan eksplisit**
- Lebih aman untuk data kompleks (misalnya string dengan spasi atau karakter khusus)
- Lebih mudah dibaca oleh developer lain

Format ini biasanya jadi pilihan utama dalam penulisan query yang rapi dan maintainable.

#### Format 2: Curly Brace Syntax (Lebih Ringkas)

Format kedua menggunakan tanda kurung kurawal `{}`.

```sql
INSERT INTO array_example (integer_array, text_array, boolean_array)
VALUES (
    '{1, 2, 3}',
    '{rose, daisy, poppy}',
    '{true, false, true}'
);
```

Penjelasan:

- Semua nilai ditulis dalam bentuk string
- PostgreSQL akan otomatis mengkonversinya menjadi array

Kelebihan:

- Lebih **singkat dan cepat ditulis**

Namun ada beberapa hal yang perlu diperhatikan:

- Karena berbentuk string, lebih rawan error jika format tidak tepat
- Untuk teks yang mengandung spasi atau karakter khusus, perlu penanganan ekstra

#### Perbandingan Singkat

- `ARRAY[...]` → lebih jelas, lebih aman, lebih direkomendasikan
- `{...}` → lebih ringkas, tapi perlu lebih hati-hati

#### Nested Arrays (Array Bertingkat)

Untuk array multidimensi, kita juga bisa menggunakan format curly brace.

```sql
-- Nested array dengan curly brace syntax
INSERT INTO array_example (nested_array)
VALUES (
    '{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}}'
);
```

Data tersebut akan disimpan sebagai array 2 dimensi (3x3), yang bisa dibayangkan seperti ini:

```
[
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9]
]
```

Artinya:

- Setiap `{}` di dalam `{}` merepresentasikan satu baris array
- Struktur ini cocok untuk data berbentuk matriks

#### Format Output Array di PostgreSQL

Hal penting yang sering membingungkan:
**PostgreSQL selalu menampilkan array dalam format curly brace `{}`**, tidak peduli bagaimana cara kita memasukkannya.

Contoh:

```sql
SELECT * FROM array_example;
```

Hasilnya:

```sql
integer_array: {1,2,3}
text_array: {rose,daisy,poppy}
nested_array: {{1,2,3},{4,5,6},{7,8,9}}
```

Penjelasan:

- Tidak ada spasi dalam output
- Format `{}` adalah **representasi standar internal PostgreSQL**

Jadi walaupun kita insert menggunakan `ARRAY[...]`, hasil query tetap akan ditampilkan dalam bentuk `{...}`.

#### Kesimpulan

- PostgreSQL menyediakan dua cara insert array:
  - `ARRAY[...]` → lebih jelas dan direkomendasikan
  - `{...}` → lebih ringkas tapi perlu hati-hati

- Untuk nested array, format `{}` sering lebih praktis
- Output array **selalu menggunakan curly brace**, karena itu adalah format standar PostgreSQL

Dengan memahami kedua format ini, kamu bisa lebih fleksibel dalam menulis query sekaligus tetap menjaga keterbacaan kode.

### d. Operator dan Fungsi untuk Arrays

#### 1. Indexing: Mengakses Element Tunggal

Salah satu operasi paling dasar saat bekerja dengan array di PostgreSQL adalah **mengakses elemen tertentu berdasarkan posisinya (index)**.

Hal yang sangat penting untuk diingat:

- PostgreSQL menggunakan **ONE-BASED INDEXING**
- Artinya, index dimulai dari **1**, bukan 0

Contoh:

```sql
SELECT text_array[1] FROM array_example;
-- Output: 'rose' (element pertama)
```

Jika mencoba mengakses index 0:

```sql
SELECT text_array[0] FROM array_example;
-- Output: NULL (tidak ada element ke-0)
```

Karena memang index 0 **tidak valid** di PostgreSQL.

Contoh lain:

```sql
SELECT text_array[3] FROM array_example;
-- Output: 'poppy' (element ketiga)
```

Perbandingan dengan JSON:

```sql
-- Native array (PostgreSQL): dimulai dari 1
SELECT ARRAY['a', 'b', 'c'][1];  -- Output: 'a'

-- JSON array: dimulai dari 0
SELECT '["a", "b", "c"]'::JSONB -> 0;  -- Output: "a"
```

Jadi, perbedaan ini penting karena sering jadi sumber bug saat berpindah antara array PostgreSQL dan JSON.

#### 2. Slicing: Mengambil Sebagian Array

Selain mengambil satu elemen, kita juga bisa mengambil **sebagian isi array (subset)** menggunakan slicing.

Contoh:

```sql
-- Mengambil dari index 1 sampai 3
SELECT text_array[1:3] FROM array_example;
-- Output: {rose,daisy,poppy}
```

Beberapa variasi slicing:

```sql
-- Dari awal sampai index 3
SELECT text_array[:3] FROM array_example;
-- Output: {rose,daisy,poppy}

-- Dari index 3 sampai akhir
SELECT text_array[3:] FROM array_example;
-- Output: {poppy}
```

Menariknya, PostgreSQL cukup “aman” terhadap range berlebih:

```sql
SELECT text_array[1:10] FROM array_example;
-- Output: {rose,daisy,poppy}
```

Walaupun kita minta sampai index 10, PostgreSQL tidak error — hanya mengembalikan elemen yang tersedia.

#### 3. Array Contains Operator: `@>`

Operator `@>` digunakan untuk mengecek apakah sebuah array **mengandung nilai tertentu**.

Contoh penggunaan:

```sql
-- Cek apakah array mengandung 'poppy'
SELECT id, text_array
FROM array_example
WHERE text_array @> ARRAY['poppy'];
```

Query ini akan mengembalikan semua baris yang memiliki `'poppy'` di dalam array-nya.

Versi shorthand:

```sql
SELECT id, text_array
FROM array_example
WHERE text_array @> '{poppy}';
```

Keduanya menghasilkan hasil yang sama.

Untuk mengecek beberapa nilai sekaligus:

```sql
SELECT id, text_array
FROM array_example
WHERE text_array @> ARRAY['rose', 'poppy'];
```

Artinya:

- Baris hanya akan dikembalikan jika mengandung **kedua nilai tersebut**

Use case umum:

- Filtering berdasarkan tag
- Mencari data dengan atribut tertentu dalam array
- Mengecek membership (apakah suatu nilai ada di dalam array)

#### 4. UNNEST: Mengubah Array Menjadi Rows

`UNNEST` adalah fungsi yang sangat penting karena mengubah array menjadi **baris-baris data (rows)**.

Contoh sederhana:

```sql
SELECT UNNEST(text_array) AS flower
FROM array_example;
```

Hasilnya:

```
flower
------
rose
daisy
poppy
```

Setiap elemen array menjadi satu baris.

Jika digabung dengan kolom lain:

```sql
SELECT
    id,
    UNNEST(text_array) AS flower
FROM array_example;
```

Hasilnya:

```
id | flower
---|-------
1  | rose
1  | daisy
1  | poppy
```

Perhatikan:

- Nilai `id` akan diulang untuk setiap elemen array

Untuk query yang lebih kompleks, biasanya digunakan bersama CTE:

```sql
WITH flowers AS (
    SELECT
        id,
        UNNEST(text_array) AS flower
    FROM array_example
)
SELECT *
FROM flowers
WHERE flower = 'poppy';
```

Query ini:

1. Mengubah array menjadi baris (via `UNNEST`)
2. Menyimpan hasilnya sementara (CTE)
3. Melakukan filtering seperti query SQL biasa

#### Kegunaan UNNEST

Fungsi ini sangat powerful karena memungkinkan kita:

- Mengubah data dari “bentuk array” ke “bentuk tabel”
- Menggunakan operasi SQL biasa (filter, join, agregasi)
- Menggabungkan dengan tabel lain
- Melakukan analisis per elemen array

#### Kesimpulan

- Indexing di PostgreSQL dimulai dari **1**, bukan 0
- Slicing memungkinkan kita mengambil sebagian array dengan fleksibel
- Operator `@>` digunakan untuk mengecek isi array
- `UNNEST` mengubah array menjadi rows sehingga bisa diproses seperti data tabel biasa

Dengan memahami operator dan fungsi ini, kamu bisa memanfaatkan array secara jauh lebih maksimal, bukan hanya sebagai tempat penyimpanan, tetapi juga sebagai bagian aktif dalam query.

### e. Kapan Menggunakan Arrays vs Tabel Terpisah?

#### Konsep Dasar: Ini Soal Design, Bukan Benar atau Salah

Dalam PostgreSQL, memilih antara menggunakan **array** atau **tabel terpisah (normalized table)** adalah keputusan desain yang sangat penting.

Tidak ada jawaban mutlak benar atau salah — semuanya tergantung pada:

- bagaimana data digunakan
- seberapa sering diakses
- kebutuhan konsistensi dan relasi

Pendekatannya selalu **case-by-case**.

#### ✅ Use Case yang Cocok untuk Arrays

##### 1. Sensor Readings atau Time-Series Data

Contoh:

```sql id="0e1l8l"
CREATE TABLE sensor_data (
    sensor_id BIGINT,
    timestamp TIMESTAMPTZ,
    readings INTEGER[]  -- Array of readings dalam 1 menit
);
```

Insert data:

```sql id="3qazvu"
INSERT INTO sensor_data VALUES
    (1, NOW(), ARRAY[20, 21, 20, 22, 21, ...]);  -- 60 values
```

Bayangkan:

- Sensor mengambil 60 data per menit
- Semua data itu disimpan dalam satu array

Kenapa ini cocok?

- Data biasanya diambil **sebagai satu paket**
- Jarang perlu akses satu per satu
- Lebih **hemat storage** (1 row vs 60 rows)
- Query lebih cepat karena tidak perlu join atau scanning banyak baris

Pendekatan ini sangat cocok untuk data yang sifatnya “batch” atau “grouped”.

##### 2. Tags atau Labels

Contoh:

```sql id="o5a7u7"
CREATE TABLE articles (
    id BIGINT PRIMARY KEY,
    title TEXT,
    tags TEXT[]  -- Array of tag names
);
```

Insert:

```sql id="7r2l8m"
INSERT INTO articles VALUES
    (1, 'PostgreSQL Tutorial', ARRAY['database', 'sql', 'tutorial']);
```

Kenapa ini cocok?

- Tags biasanya digunakan bersama-sama
- Tidak perlu tabel penghubung (junction table)
- Schema jadi lebih **sederhana dan ringan**

Namun ada trade-off:

- ❗ Tidak ada **referential integrity**
- Jika tag dihapus dari “master data”, array tidak otomatis ikut update
- Konsistensi data lebih sulit dijaga

#### ❌ Kapan Sebaiknya TIDAK Menggunakan Arrays

##### 1. Ketika Butuh Referential Integrity yang Ketat

Contoh yang bermasalah:

```sql id="6c5zlu"
CREATE TABLE posts (
    id BIGINT PRIMARY KEY,
    category_ids INTEGER[]  -- Tidak ada foreign key constraint
);
```

Masalah:

- Tidak ada jaminan bahwa `category_ids` benar-benar ada di tabel `categories`
- Data bisa menjadi tidak konsisten

Solusi yang lebih baik:

```sql id="4m6jv1"
CREATE TABLE posts (
    id BIGINT PRIMARY KEY
);

CREATE TABLE post_categories (
    post_id BIGINT REFERENCES posts(id),
    category_id BIGINT REFERENCES categories(id),
    PRIMARY KEY (post_id, category_id)
);
```

Dengan pendekatan ini:

- Relasi lebih jelas
- Integritas data terjaga oleh database

##### 2. Ketika Sering Query atau Update Elemen Individual

Jika kamu sering:

- filter berdasarkan satu elemen
- melakukan `GROUP BY` per elemen
- update satu nilai di dalam array

Maka array akan menjadi tidak efisien.

Lebih baik gunakan tabel terpisah karena:

- SQL lebih optimal untuk operasi berbasis baris
- Query lebih fleksibel

##### 3. Ketika Ukuran Array Bisa Sangat Besar

Jika array:

- terus bertambah tanpa batas (unbounded)
- berisi ratusan atau ribuan elemen

Maka:

- performa bisa menurun
- operasi menjadi lebih berat

Dalam kasus ini, struktur tabel terpisah jauh lebih scalable.

#### Decision Matrix (Panduan Praktis)

Gunakan array jika:

- Data selalu digunakan sebagai satu kesatuan
- Tidak butuh referential integrity yang ketat
- Ukuran array kecil dan terkontrol
- Ingin schema lebih sederhana
- Ada keuntungan performa dari denormalisasi

Hindari array jika:

- Perlu relasi yang ketat antar tabel
- Sering mengakses elemen secara individual
- Ukuran data bisa besar dan tidak terbatas
- Setiap elemen punya atribut tambahan
- Butuh struktur yang rapi untuk reporting atau analisis

#### Kesimpulan

Array adalah fitur yang powerful, tetapi bukan pengganti penuh dari desain relational yang baik.

Gunakan array ketika:

- data bersifat “grouped”
- aksesnya sederhana
- dan tidak membutuhkan relasi kompleks

Namun untuk sistem yang membutuhkan konsistensi tinggi, fleksibilitas query, dan skalabilitas, pendekatan **tabel terpisah (normalized)** tetap menjadi pilihan utama.

### f. Karakteristik Penting Arrays di PostgreSQL

#### Gambaran Umum

Saat bekerja dengan array di PostgreSQL, ada beberapa karakteristik penting yang perlu benar-benar dipahami. Karakteristik ini sering jadi sumber kesalahan jika tidak diperhatikan, terutama saat berpindah dari bahasa pemrograman lain atau dari JSON.

Mari kita bahas satu per satu dengan contoh agar lebih jelas.

#### 1. Type Safety

Salah satu keunggulan utama array di PostgreSQL adalah **type safety**.

Artinya:

- Semua elemen dalam array **harus memiliki tipe data yang sama**
- PostgreSQL akan secara otomatis melakukan validasi

Contoh yang salah:

```sql id="2xq9hl"
-- ❌ Error: tidak bisa mencampur tipe data
INSERT INTO array_example (integer_array)
VALUES (ARRAY[1, 'two', 3]);
```

Kenapa error?

- Karena `'two'` adalah string, sementara elemen lain adalah integer

Contoh yang benar:

```sql id="h7n3bk"
-- ✅ Semua elemen bertipe integer
INSERT INTO array_example (integer_array)
VALUES (ARRAY[1, 2, 3]);
```

Dengan adanya type safety ini:

- Data jadi lebih konsisten
- Lebih aman dari bug yang disebabkan tipe data campuran

#### 2. One-Based Indexing (Cukup Unik)

Berbeda dengan banyak bahasa pemrograman (seperti JavaScript atau Python), PostgreSQL menggunakan:

- **Index dimulai dari 1**, bukan 0

Ini sering jadi “jebakan” bagi developer.

Contoh:

```sql id="2b4q6m"
SELECT ARRAY['a', 'b', 'c'][1];  -- Output: 'a'
```

Jika mencoba index 0:

```sql id="a8d0px"
SELECT ARRAY['a', 'b', 'c'][0];  -- Output: NULL
```

Perbandingan penting:

- Array PostgreSQL → mulai dari 1
- JSON array → mulai dari 0

Jadi, kalau kamu migrasi dari JSON atau bahasa lain, bagian ini wajib diperhatikan.

#### 3. NULL Handling

Array di PostgreSQL cukup fleksibel dalam menangani `NULL`.

Ada dua kemungkinan yang berbeda:

##### a. NULL sebagai elemen di dalam array

```sql id="d2l6p1"
SELECT ARRAY[1, NULL, 3];
-- Output: {1,NULL,3}
```

Artinya:

- Array tetap ada
- Salah satu elemennya bernilai NULL

##### b. Array itu sendiri NULL

```sql id="o9r3kt"
SELECT NULL::INTEGER[];
-- Output: NULL
```

Artinya:

- Tidak ada array sama sekali (bukan array kosong)

Perbedaan ini penting:

- `{NULL}` → array dengan satu elemen NULL
- `NULL` → tidak ada array

Kesalahan memahami ini bisa berdampak pada hasil query, terutama saat filtering.

#### 4. Size Limits

Secara teori:

- PostgreSQL **tidak memiliki batas ukuran array yang kaku (hard limit)**

Namun dalam praktik:

- Semakin besar array, semakin berat performanya
- Operasi seperti search, update, atau unnest akan lebih mahal

Contoh kasus:

- Array kecil (misalnya 3–10 elemen) → sangat efisien
- Array besar (ratusan/ribuan elemen) → mulai terasa lambat

Best practice:

- Gunakan array untuk data yang **ukuran relatif kecil dan terkontrol**
- Hindari array yang terus bertambah tanpa batas

#### Kesimpulan

Beberapa karakteristik penting array di PostgreSQL:

- **Type safety** memastikan semua elemen konsisten
- **One-based indexing** berbeda dari kebanyakan sistem lain
- **NULL handling** punya dua bentuk (elemen NULL vs array NULL)
- **Tidak ada batas ukuran pasti**, tapi performa tetap perlu diperhatikan

Dengan memahami karakteristik ini, kamu bisa menggunakan array secara lebih aman, efisien, dan minim bug dalam implementasi nyata.

## 3. Hubungan Antar Konsep

Arrays di PostgreSQL melengkapi ecosystem tipe data yang sudah ada:

```
Data Storage Strategies
         |
    ┌────┴─────┐
    |          |
Structured  Semi-structured
    |          |
    |      ┌───┴────┐
    |      |        |
Discrete  JSON    Arrays
Columns           |
    |          Multiple
    |          values,
Foreign    same type
Keys       |
    |      └─── Trade-off:
Junction       Simplicity
Tables         vs
    |          Normalization
    └──────────┘
```

**Alur keputusan storage:**

1. **Start with normalized structure** → Foreign keys + junction tables
2. **Identify repeated patterns** → Candidate untuk arrays atau JSON
3. **Evaluate access patterns:**
   - Sering akses bersama? → Arrays
   - Perlu flexibility? → JSON
   - Butuh referential integrity? → Stay normalized
4. **Consider trade-offs** dan pilih yang best fit

**Arrays vs JSON vs Normalized:**

| Aspect                    | Arrays       | JSON         | Normalized Tables |
| ------------------------- | ------------ | ------------ | ----------------- |
| **Type safety**           | ✅ Strong    | ❌ Weak      | ✅ Strong         |
| **Referential integrity** | ❌ No        | ❌ No        | ✅ Yes            |
| **Query flexibility**     | 🟡 Good      | 🟡 Good      | ✅ Excellent      |
| **Schema simplicity**     | ✅ Simple    | ✅ Simple    | ❌ Complex        |
| **Performance (bulk)**    | ✅ Fast      | ✅ Fast      | 🟡 Slower         |
| **Indexing**              | ✅ Supported | ✅ Supported | ✅ Native         |

**Kombinasi strategies:**

```sql
-- Hybrid approach: best of both worlds
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name TEXT,

    -- Normalized: strict integrity
    category_id BIGINT REFERENCES categories(id),

    -- Arrays: simple repeated values
    image_urls TEXT[],

    -- JSON: flexible attributes
    specifications JSONB
);
```

**Operator ecosystem:**

```
Array Operations:
    |
    ├─ Indexing ([n])        → Access single element
    ├─ Slicing ([m:n])       → Get subset
    ├─ Contains (@>)         → Membership check
    ├─ UNNEST()              → Convert to rows
    ├─ array_length()        → Get size
    ├─ array_append()        → Add element
    ├─ array_cat()           → Concatenate
    └─ ... many more         → (akan dibahas di querying section)
```

## 4. Kesimpulan

- PostgreSQL mendukung **arrays sebagai tipe data standalone** yang sangat powerful
- Arrays bisa dideklarasikan dengan `TYPE[]` atau `ARRAY` keyword
- **Two formats untuk insert**: `ARRAY[...]` constructor atau `{...}` curly brace syntax
- **⚠️ PostgreSQL arrays menggunakan ONE-BASED INDEXING** (dimulai dari 1, bukan 0)
- **Key operators dan functions:**
  - **Indexing** `[n]` → akses element
  - **Slicing** `[m:n]` → ambil subset
  - **Contains** `@>` → check membership
  - **UNNEST** → convert array ke rows (sangat powerful!)
- **Kapan menggunakan arrays:**
  - ✅ Data selalu diakses bersama-sama
  - ✅ Schema simplicity lebih penting
  - ✅ Array size bounded dan reasonable
  - ❌ Hindari jika butuh strict referential integrity
  - ❌ Hindari jika sering query individual elements
- **Trade-offs:**
  - 👍 Simplifikasi schema (no junction tables)
  - 👍 Performance untuk bulk access
  - 👎 Kehilangan referential integrity
  - 👎 Lebih sulit maintain consistency
- Arrays sangat berguna untuk **sensor data, tags, lists of values** yang naturally grouped together
- PostgreSQL menyediakan **robust functions dan operators** untuk manipulasi arrays (akan dibahas lebih detail di section querying dan indexing)

**Template use case yang cocok:**

```sql
-- Sensor/IoT data
CREATE TABLE sensor_readings (
    sensor_id BIGINT,
    timestamp TIMESTAMPTZ,
    temperature_readings NUMERIC[],  -- Array of readings
    PRIMARY KEY (sensor_id, timestamp)
);

-- Tags/Labels
CREATE TABLE blog_posts (
    id BIGINT PRIMARY KEY,
    title TEXT,
    content TEXT,
    tags TEXT[],                     -- Array of tag names
    created_at TIMESTAMPTZ
);

-- Multiple URLs/Paths
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name TEXT,
    image_urls TEXT[],               -- Array of image URLs
    document_paths TEXT[]            -- Array of file paths
);
```
