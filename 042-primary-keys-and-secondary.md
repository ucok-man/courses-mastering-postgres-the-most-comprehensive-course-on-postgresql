# 📝 Primary Keys vs Secondary Indexes (di PostgreSQL)

## 1. Ringkasan Singkat

Video ini membahas perbedaan mendasar antara **primary key** dan **secondary index** dalam konteks database, khususnya PostgreSQL. Untuk memahami konsep ini dengan baik, video membandingkan cara MySQL dan PostgreSQL menyimpan data di disk — karena keduanya memiliki pendekatan yang sangat berbeda. Kesimpulan utamanya: **di PostgreSQL, semua index adalah secondary index**, termasuk index yang dibuat otomatis oleh primary key.

---

## 2. Konsep Utama

### a. Clustered Index (Konsep di MySQL)

#### Memahami Peran Primary Key sebagai Clustered Index

Di MySQL, saat kamu mendefinisikan sebuah **primary key**, sebenarnya kamu tidak hanya menentukan identitas unik untuk setiap baris — kamu juga secara otomatis membuat **clustered index**.

Clustered index ini sangat penting karena menentukan **bagaimana data disimpan secara fisik di dalam disk**. Jadi bukan sekadar struktur tambahan, tapi benar-benar menjadi “kerangka utama” dari tabel itu sendiri.

#### Tabel di MySQL = B-Tree Index

Salah satu konsep kunci yang sering membingungkan di awal adalah:

> Di MySQL, **tabel itu sendiri adalah sebuah index**, tepatnya berupa struktur **B-tree**.

B-tree adalah struktur data berbentuk pohon yang dirancang untuk efisiensi pencarian, penyisipan, dan penghapusan data dalam jumlah besar.

Struktur sederhananya seperti ini:

```
         [Root Node]
        /            \
  [Internal]      [Internal]
   /      \        /      \
[Leaf]  [Leaf]  [Leaf]  [Leaf]
 ↑ Leaf node menyimpan SELURUH baris data (entire row)
```

Mari kita pahami bagian-bagiannya:

- **Root node**: titik awal pencarian
- **Internal node**: berisi pointer untuk navigasi ke node berikutnya
- **Leaf node**: bagian paling bawah, dan di sinilah **data asli (seluruh baris)** disimpan

Inilah ciri khas clustered index di MySQL:
👉 **Seluruh isi tabel disimpan langsung di leaf node B-tree**

Artinya, ketika kamu melakukan query berdasarkan primary key, database bisa langsung menemukan data tanpa perlu “loncat” ke struktur lain.

#### Kenapa Disebut “Clustered”?

Disebut **clustered** karena data di dalam tabel disusun dan “dikelompokkan” (clustered) berdasarkan urutan primary key.

Contohnya:

Jika primary key adalah `id`, maka data di disk akan disusun seperti:

```
id = 1 → id = 2 → id = 3 → id = 4 → ...
```

Urutan ini bukan kebetulan, tapi memang mengikuti struktur B-tree tadi.

#### Secondary Index di MySQL

Selain clustered index, ada juga yang disebut **secondary index** — yaitu semua index lain selain primary key.

Perbedaannya cukup penting:

- **Clustered index (primary key)**
  → Menyimpan **seluruh baris data**

- **Secondary index**
  → Hanya menyimpan:
  - nilai kolom yang diindex
  - - **primary key sebagai pointer**

Jadi ketika kamu melakukan query menggunakan secondary index, prosesnya kira-kira seperti ini:

1. Cari nilai di secondary index
2. Dapatkan primary key dari hasil tersebut
3. Gunakan primary key untuk mencari data lengkap di clustered index

Proses ini sering disebut sebagai **double lookup**.

#### Contoh Sederhana

Misalnya ada tabel:

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);
```

Lalu kamu membuat secondary index:

```sql
CREATE INDEX idx_email ON users(email);
```

Saat menjalankan query:

```sql
SELECT * FROM users WHERE email = 'user@example.com';
```

Yang terjadi:

1. MySQL mencari `'user@example.com'` di index `idx_email`
2. Ditemukan primary key, misalnya `id = 5`
3. MySQL masuk ke clustered index untuk mengambil seluruh data baris dengan `id = 5`

#### Inti yang Perlu Diingat

- Clustered index menentukan **struktur fisik tabel**
- Di MySQL, **tabel = clustered index (B-tree)**
- Leaf node menyimpan **seluruh data**
- Secondary index hanya menyimpan **referensi ke primary key**, bukan data lengkap

Dengan memahami ini, kamu akan lebih mudah mengerti kenapa desain primary key dan index sangat berpengaruh terhadap performa query di MySQL.

### b. Heap dan Secondary Index (Konsep di PostgreSQL)

#### Cara PostgreSQL Menyimpan Data

Berbeda dengan MySQL, PostgreSQL menggunakan pendekatan yang lebih fleksibel dalam menyimpan data.

Di PostgreSQL, data tidak disusun berdasarkan index tertentu. Sebaliknya, semua baris disimpan dalam sebuah struktur yang disebut **heap**.

Heap ini bisa dibayangkan sebagai:

> “tumpukan besar data tanpa urutan khusus”

Artinya:

- Data tidak diurutkan berdasarkan primary key atau kolom lain
- Urutan penyimpanan biasanya mengikuti waktu insert (meskipun bisa berubah karena update/delete)

#### Konsekuensi: Tidak Ada Clustered Index

Karena data tidak diorganisir berdasarkan index tertentu, maka:

- PostgreSQL **tidak memiliki clustered index secara default**
- Semua index yang dibuat adalah **secondary index**

Ini berbeda dengan MySQL, di mana primary key langsung menentukan struktur fisik data.

#### Peran Index di PostgreSQL

Meskipun data disimpan di heap, PostgreSQL tetap menggunakan index (biasanya berbentuk B-tree) untuk mempercepat pencarian.

Namun, cara kerjanya sedikit berbeda.

Index di PostgreSQL **tidak menyimpan seluruh data baris**, melainkan:

- nilai kolom yang diindex
- - pointer ke lokasi data di heap

Pointer ini disebut **TID (Tuple ID)**, yaitu alamat fisik dari baris di dalam heap.

#### Alur Pencarian Data

Mari kita lihat alurnya lewat contoh query:

```sql
SELECT * FROM users WHERE email = 'a@b.com';
```

Proses yang terjadi di PostgreSQL:

1. **Traversal index**
   Database menelusuri B-tree index untuk mencari nilai `'a@b.com'`

2. **Menemukan TID**
   Dari index, PostgreSQL mendapatkan pointer, misalnya `(0,1)`

3. **Akses heap**
   Menggunakan TID tersebut, PostgreSQL “melompat” ke heap untuk mengambil data lengkap

Ilustrasinya:

```id="zj24zc"
    [Index B-tree]          [Heap (data aktual)]
    email → TID             ┌─────────────────┐
    ─────────────           │ Row 1: id=1, ... │
    a@b.com → (0,1) ──────► │ Row 2: id=2, ... │ ◄── data ditemukan di sini
    b@c.com → (0,3)         │ Row 3: id=3, ... │
                            └─────────────────┘
```

Jadi, berbeda dengan MySQL yang bisa langsung mengambil data dari index, PostgreSQL hampir selalu melakukan dua langkah:

- cari di index
- ambil data di heap

Proses ini sering disebut sebagai **index scan + heap fetch**.

#### Kelebihan dan Implikasi

Pendekatan heap ini memberikan fleksibilitas, misalnya:

- Tidak terikat pada satu urutan fisik (lebih bebas untuk update data)
- Bisa memiliki banyak index tanpa memengaruhi struktur penyimpanan utama

Namun, ada konsekuensi:

- Ada tambahan langkah saat membaca data (harus ke heap)
- Bisa lebih lambat dibanding clustered index dalam beberapa kasus

#### Pengecualian: Covering Index

Ada satu kondisi khusus di mana PostgreSQL **tidak perlu mengakses heap**, yaitu ketika menggunakan **covering index**.

Konsepnya sederhana:

- Jika semua kolom yang dibutuhkan oleh query sudah tersedia di dalam index
- Maka PostgreSQL bisa langsung mengambil hasil dari index saja

Contoh:

```sql
CREATE INDEX idx_email_name ON users(email, name);
```

Lalu query:

```sql
SELECT name FROM users WHERE email = 'a@b.com';
```

Karena kolom `email` dan `name` sudah ada di index:

- PostgreSQL tidak perlu ke heap
- Query bisa lebih cepat karena hanya membaca index

#### Inti yang Perlu Diingat

- PostgreSQL menyimpan data di **heap (tidak terurut)**
- Semua index adalah **secondary index**
- Index menyimpan **nilai + TID (pointer ke heap)**
- Query biasanya melibatkan:
  - traversal index
  - lalu akses heap

- **Covering index** bisa menghindari akses ke heap jika semua data sudah tersedia di index

Memahami perbedaan ini penting, karena strategi optimasi di PostgreSQL sering kali berfokus pada bagaimana meminimalkan akses ke heap dan memaksimalkan efisiensi penggunaan index.

### c. Primary Key di PostgreSQL

#### Apakah Primary Key Masih Spesial?

Di PostgreSQL, kita sudah tahu bahwa semua index pada dasarnya adalah **secondary index** karena data utama disimpan di heap.

Namun, bukan berarti **primary key** kehilangan perannya. Justru, primary key tetap punya “keistimewaan” tersendiri — bisa dibilang sebagai:

> **secondary index++ (versi yang lebih kuat dari index biasa)**

Artinya, primary key bukan sekadar index untuk mempercepat pencarian, tapi juga membawa aturan (constraint) penting untuk menjaga integritas data.

#### Apa yang Terjadi Saat Mendeklarasikan Primary Key?

Ketika kamu menuliskan `PRIMARY KEY` pada sebuah kolom, PostgreSQL secara otomatis melakukan tiga hal sekaligus:

##### 1. Menambahkan NOT NULL

Kolom tersebut **tidak boleh bernilai `NULL`**.

Kenapa penting?
Karena primary key digunakan sebagai identitas unik setiap baris. Kalau `NULL` diperbolehkan, maka identitasnya jadi tidak jelas.

##### 2. Menambahkan Constraint UNIQUE

Setiap nilai dalam kolom primary key harus **unik (tidak boleh duplikat)**.

Ini memastikan:

- Tidak ada dua baris dengan identitas yang sama
- Data tetap konsisten dan bisa di-refer dengan aman (misalnya untuk relasi antar tabel)

##### 3. Membuat Index Secara Otomatis

PostgreSQL akan langsung membuat index (biasanya berbasis B-tree) untuk kolom tersebut.

Jadi kamu tidak perlu lagi membuat index manual untuk primary key — semuanya sudah diurus otomatis.

#### Contoh DDL dan Apa yang Terjadi di Baliknya

Mari lihat contoh sederhana:

```sql
-- Kamu hanya menulis ini
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT
);
```

Sekilas terlihat sederhana, tapi di balik layar PostgreSQL melakukan beberapa hal penting:

1. Mengubah kolom `id` menjadi **NOT NULL**
2. Menambahkan constraint **UNIQUE** pada `id`
3. Membuat **index B-tree** untuk kolom `id`

Selain itu, `SERIAL` juga berarti:

- PostgreSQL membuat sequence otomatis
- Nilai `id` akan bertambah sendiri setiap insert

#### Kenapa Disebut “Secondary Index++”?

Kalau dibandingkan dengan index biasa:

- **Index biasa**
  → hanya membantu mempercepat pencarian

- **Primary key**
  → mempercepat pencarian **+ menjaga integritas data**

Jadi primary key bukan cuma soal performa, tapi juga soal aturan data yang wajib dipatuhi.

#### Inti yang Perlu Diingat

- Di PostgreSQL, primary key tetap berupa **secondary index**
- Tapi memiliki tambahan:
  - **NOT NULL**
  - **UNIQUE**
  - **index otomatis**

- Primary key berfungsi sebagai:
  - identitas unik baris
  - sekaligus alat optimasi query

Memahami ini penting, karena desain primary key yang baik tidak hanya berdampak pada performa, tapi juga memastikan struktur data kamu tetap rapi dan konsisten.

### d. Aturan: Satu Primary Key, Banyak Secondary Index

#### Kenapa Primary Key Hanya Satu?

Di PostgreSQL (dan juga di sebagian besar database relasional), setiap tabel hanya boleh memiliki **satu primary key**.

Alasannya cukup mendasar:

- **Primary key adalah identitas utama** untuk setiap baris data
- Karena sifatnya “utama” (_primary_), maka tidak masuk akal jika ada lebih dari satu identitas utama dalam satu tabel

Primary key bisa berupa:

- satu kolom (misalnya `id`)
- atau kombinasi beberapa kolom (disebut **composite primary key**)

Tapi tetap, secara konsep, itu dihitung sebagai **satu primary key**.

#### Secondary Index: Bebas Membuat Banyak

Berbeda dengan primary key, kamu **boleh membuat banyak secondary index** sesuai kebutuhan.

Secondary index ini digunakan untuk:

- mempercepat query
- mengoptimalkan pencarian berdasarkan kolom tertentu
- mendukung kondisi `WHERE`, `ORDER BY`, atau `JOIN`

Karena sifatnya opsional dan spesifik kebutuhan, jumlahnya tidak dibatasi secara konsep (meskipun secara praktis tetap perlu dikontrol agar tidak berlebihan).

#### Contoh Implementasi

Mari kita lihat contoh tabel:

```sql id="8psqg1"
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,           -- primary key (hanya boleh satu)
    user_id INT NOT NULL,
    status TEXT,
    created_at TIMESTAMPTZ
);
```

Di sini:

- `id` adalah **primary key**
- PostgreSQL otomatis:
  - memberi constraint NOT NULL
  - memastikan UNIQUE
  - membuat index untuk kolom tersebut

Sekarang kita tambahkan beberapa secondary index:

```sql id="jz4l9n"
CREATE INDEX idx_orders_user_id    ON orders(user_id);
CREATE INDEX idx_orders_status     ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at);
```

#### Bagaimana Index Ini Digunakan?

Misalnya kamu menjalankan query:

```sql id="g9n4lz"
SELECT * FROM orders WHERE user_id = 10;
```

- PostgreSQL bisa menggunakan `idx_orders_user_id` untuk mempercepat pencarian

Atau:

```sql id="9a2wqk"
SELECT * FROM orders WHERE status = 'shipped';
```

- Akan menggunakan `idx_orders_status`

Dan untuk query berbasis waktu:

```sql id="xv7m2c"
SELECT * FROM orders ORDER BY created_at DESC;
```

- Bisa memanfaatkan `idx_orders_created_at`

#### Catatan Penting: Jangan Terlalu Banyak Index

Walaupun “boleh banyak”, bukan berarti “sebanyak mungkin”.

Setiap index punya biaya:

- memperlambat operasi `INSERT`, `UPDATE`, dan `DELETE` (karena index juga harus ikut diperbarui)
- menambah penggunaan storage

Jadi praktik terbaiknya:

- buat index hanya untuk kolom yang sering digunakan dalam query
- evaluasi performa secara berkala

#### Inti yang Perlu Diingat

- Satu tabel hanya punya **satu primary key**
- Primary key = identitas utama baris
- Secondary index bisa dibuat **sebanyak yang dibutuhkan**
- Gunakan secondary index untuk mengoptimalkan query, tapi tetap bijak agar tidak membebani sistem

Dengan memahami aturan ini, kamu bisa mulai merancang struktur tabel yang seimbang antara integritas data dan performa query.

### e. Perbandingan: Bisa NOT NULL + UNIQUE, Tapi Bukan Primary Key?

#### Apakah UNIQUE + NOT NULL Sama dengan Primary Key?

Di PostgreSQL, kamu bisa membuat sebuah kolom memiliki constraint **`UNIQUE`** dan **`NOT NULL`**, sehingga secara perilaku terlihat mirip dengan primary key.

Namun, penting untuk dipahami:

> Meskipun secara teknis mirip, **itu tetap bukan primary key**

Perbedaannya bukan pada kemampuan teknis semata, tapi lebih ke **makna (semantik)** dan **peran dalam desain database**.

#### Perbedaan dari Sisi Konsep

Mari kita bedakan secara jelas:

- **Primary Key**
  - Identitas utama setiap baris
  - Hanya boleh satu per tabel
  - Digunakan sebagai referensi utama (misalnya untuk relasi antar tabel / foreign key)

- **UNIQUE + NOT NULL (Secondary Index)**
  - Hanya memastikan nilai unik dan tidak kosong
  - Bisa ada lebih dari satu di tabel
  - Tidak dianggap sebagai “identitas utama”

Jadi walaupun dua kolom sama-sama unik dan tidak null, hanya satu yang “diangkat” sebagai primary key.

#### Contoh Kasus: Email sebagai Unique, Bukan Primary Key

Misalnya kamu punya tabel `users`, dan ingin memastikan bahwa email:

- tidak boleh kosong
- tidak boleh duplikat

Kamu bisa melakukannya seperti ini:

```sql id="t6azx1"
-- Membuat unique index untuk email
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Menambahkan constraint NOT NULL
ALTER TABLE users ALTER COLUMN email SET NOT NULL;
```

Dengan ini:

- Tidak akan ada dua user dengan email yang sama
- Semua user wajib punya email

#### Tapi Kenapa Tidak Dijadikan Primary Key?

Pertanyaannya bagus: kalau sudah unik dan tidak null, kenapa tidak sekalian jadi primary key?

Jawabannya biasanya terkait desain:

- **Primary key sebaiknya stabil dan sederhana**
  Email bisa berubah (user ganti email), sedangkan primary key idealnya tidak berubah

- **Primary key sering digunakan sebagai referensi**
  Lebih aman menggunakan ID (misalnya `id SERIAL`) daripada email

- **Konsistensi desain**
  Banyak sistem menggunakan pola:
  - `id` sebagai primary key
  - field lain (seperti email) sebagai unique constraint tambahan

#### Inti yang Perlu Diingat

- `UNIQUE + NOT NULL` bisa meniru sebagian perilaku primary key
- Tapi tetap **bukan pengganti primary key**
- Perbedaan utamanya ada di:
  - peran sebagai identitas utama
  - penggunaan dalam relasi antar tabel

- Gunakan:
  - **primary key** untuk identitas utama
  - **unique constraint/index** untuk aturan tambahan seperti email unik

Dengan memahami perbedaan ini, kamu bisa merancang schema database yang lebih konsisten, fleksibel, dan mudah dikembangkan ke depannya.

## 3. Hubungan Antar Konsep

Konsep-konsep di atas membentuk alur pemahaman yang saling berkaitan:

1. **MySQL vs PostgreSQL** → Perbedaan cara penyimpanan data (clustered index vs heap) menjadi fondasi untuk memahami mengapa istilah "secondary index" di Postgres berbeda makna dari di MySQL.

2. **Heap → semua index adalah secondary** → Karena PostgreSQL menyimpan data di heap tanpa urutan tertentu, tidak ada satu index pun yang "menjadi" tabel itu sendiri. Semua index hanya menyimpan pointer ke heap.

3. **Secondary index + aturan khusus → Primary Key** → Primary key bukan jenis index yang benar-benar berbeda secara teknis, melainkan secondary index yang dilengkapi dengan constraint `NOT NULL`, `UNIQUE`, dan dibuat secara otomatis — menjadikannya "secondary index plus-plus."

4. **Covering index (pengecualian)** → Satu-satunya skenario di mana PostgreSQL tidak perlu bolak-balik ke heap adalah covering index, yang akan dibahas lebih lanjut di topik lanjutan.

---

## 4. Kesimpulan

- Di **MySQL**, primary key = clustered index = cara data disimpan di disk. Tabel _adalah_ index itu sendiri.
- Di **PostgreSQL**, data disimpan di **heap** (tidak berurutan), sehingga **semua index bersifat secondary** — termasuk index yang dibuat oleh primary key.
- **Primary key** di PostgreSQL adalah secondary index dengan tiga keistimewaan otomatis: `NOT NULL`, `UNIQUE`, dan index dibuat sendiri.
- Hanya **satu primary key** per tabel, tetapi secondary index bisa sebanyak yang dibutuhkan.
- Setiap pencarian via index di PostgreSQL (kecuali covering index) selalu melibatkan dua langkah: **traversal index → fetch ke heap**.
