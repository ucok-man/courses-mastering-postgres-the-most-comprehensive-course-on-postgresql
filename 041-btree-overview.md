# 📝 Gambaran Umum B-Tree (B-Tree Overview)

## 1. Ringkasan Singkat

Video ini membahas struktur data **B-Tree**, yaitu jenis index yang paling umum digunakan dalam database. Materi mencakup bagaimana B-Tree divisualisasikan, cara database melakukan pencarian (traversal) menggunakan struktur ini, serta mengapa B-Tree jauh lebih efisien dibandingkan dengan melakukan pemindaian seluruh tabel (table scan). Pemahaman ini sangat berguna untuk membangun intuisi kapan sebuah index akan bermanfaat dan kapan tidak.

---

## 2. Konsep Utama

### a. Apa Itu B-Tree?

#### Pengertian Dasar

**B-Tree** adalah salah satu struktur data berbentuk pohon (_tree_) yang digunakan oleh database untuk menyimpan **index**. Tujuan utama dari index ini adalah mempercepat proses pencarian data tanpa harus memindai seluruh isi tabel.

Berbeda dengan struktur tree sederhana, B-Tree dirancang agar tetap **seimbang (balanced)**. Artinya, kedalaman setiap cabang relatif sama, sehingga waktu pencarian tetap konsisten dan efisien meskipun jumlah data bertambah banyak.

Dalam dunia database relasional, B-Tree adalah **jenis index yang paling umum digunakan** karena fleksibel dan cocok untuk berbagai kebutuhan query.

#### Kenapa B-Tree Efisien?

B-Tree bekerja dengan cara menyimpan data dalam bentuk yang **terurut (sorted)**. Dengan begitu, database bisa melakukan pencarian menggunakan teknik seperti _binary search_, yang jauh lebih cepat dibandingkan pencarian linear.

Struktur ini memungkinkan operasi berikut berjalan dengan cepat:

- **Pencarian (search)**
- **Penambahan data (insert)**
- **Penghapusan data (delete)**

Semua operasi ini umumnya memiliki kompleksitas waktu **O(log n)**, yang artinya tetap cepat walaupun data sangat besar.

#### Jenis Query yang Cocok untuk B-Tree

B-Tree sangat optimal untuk dua jenis operasi yang sering digunakan dalam query SQL:

1. **Strict equality (pencarian nilai yang sama persis)**
   Contoh:

   ```sql
   SELECT * FROM users WHERE name = 'Jennifer';
   ```

   Database bisa langsung menemukan lokasi data tanpa harus membaca seluruh tabel.

2. **Range queries (pencarian dalam rentang nilai)**
   Contoh:

   ```sql
   SELECT * FROM users WHERE age BETWEEN 20 AND 30;
   ```

   Karena data tersusun rapi, database bisa langsung mengambil rentang nilai yang diminta secara efisien.

#### Ilustrasi Konteks

Bayangkan kita punya tabel `users` seperti ini:

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name TEXT,
  email TEXT
);
```

Lalu kita membuat index pada kolom `name`:

```sql
CREATE INDEX idx_users_name ON users(name);
```

Dengan adanya index B-Tree ini:

- Database tidak perlu lagi mengecek satu per satu baris saat mencari nama tertentu
- Sistem cukup menavigasi struktur tree untuk langsung menuju data yang dicari
- Proses pencarian jadi jauh lebih cepat, terutama saat jumlah data sudah besar

#### Intuisi Sederhana

Kalau tanpa index, mencari data itu seperti membaca buku dari halaman pertama sampai akhir.

Dengan B-Tree index, itu seperti menggunakan **daftar isi atau indeks di belakang buku** — kamu langsung lompat ke halaman yang relevan tanpa membuang waktu.

Itulah kenapa B-Tree menjadi fondasi penting dalam optimasi performa database.

### b. Struktur B-Tree

#### Gambaran Umum

Struktur **B-Tree** tersusun secara hierarkis seperti pohon, tetapi dengan aturan khusus yang membuatnya tetap seimbang dan efisien untuk pencarian. Secara umum, B-Tree terdiri dari tiga bagian utama: **root node**, **interior node**, dan **leaf node**. Masing-masing punya peran yang berbeda dalam proses navigasi data.

Untuk memahami cara kerjanya, bayangkan seperti sistem navigasi bertingkat: dari petunjuk utama di atas, lalu mengarah ke kategori yang lebih spesifik, hingga akhirnya menemukan data yang dicari.

#### 🔝 Root Node (Simpul Akar)

Root node adalah titik awal dari seluruh proses pencarian.

- Letaknya selalu di **paling atas** dari struktur pohon.
- Berisi beberapa **nilai kunci (keys)** yang berfungsi sebagai penunjuk arah.
- Saat database melakukan pencarian, langkah pertama selalu dimulai dari sini.

Contohnya, jika root node berisi:

```
[ Isaac | Steve ]
```

Artinya:

- Jika nilai yang dicari **kurang dari "Isaac"**, pencarian diarahkan ke cabang kiri.
- Jika berada **di antara "Isaac" dan "Steve"**, diarahkan ke cabang tengah.
- Jika **lebih dari "Steve"**, diarahkan ke cabang kanan.

Root node ini membantu mempersempit pencarian sejak langkah pertama.

#### 🌿 Interior Node (Simpul Dalam)

Interior node berada di antara root node dan leaf node. Fungsinya mirip dengan root node, tetapi bekerja di level yang lebih dalam.

- Berisi **nilai kunci tambahan** untuk memperjelas arah pencarian.
- Bertindak sebagai “filter lanjutan” setelah dari root.
- Bisa memiliki beberapa cabang, tergantung jumlah key yang dimiliki.

Contoh:

```
[ Simon ]
```

Node ini membagi data menjadi dua bagian:

- Semua nilai **lebih kecil dari "Simon"** akan berada di kiri
- Semua nilai **lebih besar dari "Simon"** akan berada di kanan

Dengan adanya interior node, proses pencarian tidak langsung lompat ke data, tapi dipersempit bertahap hingga sangat spesifik.

#### 🍃 Leaf Node (Simpul Daun)

Leaf node adalah tujuan akhir dari pencarian dalam B-Tree.

- Terletak di **paling bawah** dari struktur pohon.
- Berisi **data yang sudah diindeks**, biasanya berupa nilai kolom yang di-index.
- Disusun secara **terurut (sorted)** dari kiri ke kanan.
- Setiap entry juga menyimpan **pointer (CTID)** yang mengarah ke lokasi data sebenarnya di dalam tabel.

Contohnya:

```
[Aaron | ... | Isaac]
[Jennifer | ... | Simon]
[Steve | Taylor | Virginia]
```

Beberapa hal penting dari leaf node:

- Urutan data memungkinkan **range query** berjalan cepat
- Pointer (CTID) memungkinkan database mengambil data lengkap dari tabel utama (_heap table_) tanpa duplikasi

#### Diagram Struktur B-Tree

Berikut gambaran sederhana struktur B-Tree:

```
                    [ Isaac | Steve ]          <-- Root Node
                   /        |        \
             [...]       [Simon]      [Taylor|...]   <-- Interior Nodes
            /                \              \
[Aaron|...|Isaac] [Jennifer|...|Simon] [Steve|Taylor|Virginia] <-- Leaf Nodes
```

#### Cara Membaca Struktur Ini

Jika kita ingin mencari, misalnya, `"Jennifer"`:

1. Mulai dari root node `[Isaac | Steve]`
   - "Jennifer" berada di antara "Isaac" dan "Steve" → turun ke cabang tengah

2. Masuk ke interior node `[Simon]`
   - "Jennifer" lebih kecil dari "Simon" → ambil cabang kiri

3. Sampai di leaf node `[Jennifer | ... | Simon]`
   - Data ditemukan di sini
   - Gunakan CTID untuk mengambil baris lengkap dari tabel

#### Intuisi Sederhana

Struktur ini mirip seperti mencari kata di kamus:

- Root node = pembagian awal berdasarkan huruf besar
- Interior node = pembagian lebih spesifik
- Leaf node = halaman berisi kata sebenarnya

Karena semua data di leaf node tersusun rapi, pencarian tidak hanya cepat untuk satu nilai, tapi juga sangat efisien untuk rentang nilai sekaligus.

### c. Traversal B-Tree (Proses Pencarian)

#### Gambaran Umum

Traversal pada **B-Tree** adalah proses pencarian data dengan cara menelusuri struktur pohon dari atas ke bawah. Proses ini selalu dimulai dari **root node**, lalu bergerak ke **interior node**, hingga akhirnya mencapai **leaf node** tempat data berada.

Kunci dari proses ini adalah **perbandingan nilai (comparison)**. Di setiap node, database akan membandingkan nilai yang dicari dengan nilai kunci yang ada, lalu menentukan jalur mana yang harus diambil berikutnya.

Karena struktur B-Tree sudah terurut dan seimbang, proses ini bisa dilakukan dengan sangat efisien tanpa perlu mengecek seluruh data.

#### Contoh 1: Mencari nama `Jennifer`

Mari kita ikuti langkah pencariannya secara bertahap:

1. **Mulai dari root node**
   Root berisi:

   ```
   [Isaac | Steve]
   ```

   Kita bandingkan:
   - "Jennifer" berada **di antara "Isaac" dan "Steve"**

   Artinya, kita ambil **jalur tengah**.

2. **Masuk ke interior node**
   Node ini berisi:

   ```
   [Simon]
   ```

   Bandingkan lagi:
   - "Jennifer" < "Simon"

   Maka kita ambil **jalur kiri**.

3. **Masuk ke leaf node**
   Di sini kita menemukan:

   ```
   [Jennifer]
   ```

   Data ditemukan.

4. **Ambil data lengkap dari tabel**
   Leaf node tidak menyimpan seluruh baris data, tetapi menyimpan **CTID (pointer)**.
   CTID ini digunakan untuk mengambil data lengkap dari **heap table** (tabel utama).

Alur lengkapnya bisa diringkas seperti ini:

```
Root:     [Isaac | Steve]
             ↓ (Jennifer di tengah)
Interior: [Simon]
             ↓ (Jennifer < Simon, kiri)
Leaf:     [Jennifer] → CTID → ambil baris dari tabel
```

#### Contoh 2: Mencari nama `Taylor`

Sekarang kita coba kasus lain:

1. **Mulai dari root node**

   ```
   [Isaac | Steve]
   ```

   Bandingkan:
   - "Taylor" > "Steve"

   Maka kita ambil **jalur kanan**.

2. **Masuk ke interior node**

   ```
   [Taylor]
   ```

   Bandingkan:
   - "Taylor" = "Taylor"

   Dalam struktur B-Tree, kondisi ini biasanya mengarahkan kita ke **cabang tertentu (umumnya kanan)** untuk menemukan data aktual.

3. **Masuk ke leaf node**

   ```
   [Taylor]
   ```

   Data ditemukan.

4. **Gunakan CTID untuk mengambil data lengkap**
   Sama seperti sebelumnya, pointer digunakan untuk mengambil baris asli dari tabel.

Ringkasan alurnya:

```
Root:     [Isaac | Steve]
             ↓ (Taylor > Steve, kanan)
Interior: [Taylor]
             ↓ (Taylor = Taylor, kanan)
Leaf:     [Taylor] → CTID → ambil baris dari tabel
```

#### Intuisi Sederhana

Traversal B-Tree ini mirip seperti mengikuti petunjuk arah:

- Root node memberi petunjuk awal
- Interior node mempersempit pilihan
- Leaf node adalah tujuan akhir

Setiap langkah selalu mengurangi ruang pencarian, sehingga prosesnya cepat dan efisien, bahkan untuk dataset yang sangat besar.

### d. Index vs. Table Scan

#### Gambaran Perbedaan

Dalam database, ada dua cara utama untuk mencari data:

| Metode             | Cara Kerja                                              | Efisiensi                       |
| ------------------ | ------------------------------------------------------- | ------------------------------- |
| **Index (B-Tree)** | Traversal tree → langsung ke baris yang dicari via CTID | Sangat efisien untuk data besar |
| **Table Scan**     | Membaca semua baris satu per satu (brute force)         | Lambat untuk data besar         |

Perbedaan utamanya terletak pada **berapa banyak data yang harus diperiksa** untuk menemukan hasil.

#### Cara Kerja Table Scan

Tanpa index, database tidak punya “petunjuk arah”. Jadi satu-satunya cara adalah membaca data **baris demi baris** dari awal sampai akhir.

Contohnya:

```sql
SELECT * FROM users WHERE name = 'Jennifer';
```

Jika tidak ada index pada kolom `name`, database akan:

- Membuka heap table
- Membaca setiap baris
- Mengecek apakah `name = 'Jennifer'`
- Mengulang sampai semua baris diperiksa atau data ditemukan

Secara konsep, ini seperti:

```id="jlt3pg"
FOR EACH row IN heap_table:
    IF row.name == 'Jennifer':
        RETURN row  -- harus cek semua baris!
```

Masalahnya, semakin besar tabel, semakin lama proses ini. Untuk tabel dengan jutaan atau miliaran baris, ini jelas tidak efisien.

#### Cara Kerja Index (B-Tree)

Dengan adanya index B-Tree, database tidak perlu membaca semua data. Sebagai gantinya, database akan:

1. Menelusuri struktur B-Tree (dari root → interior → leaf)
2. Menemukan nilai yang dicari dalam beberapa langkah saja
3. Menggunakan **CTID (pointer)** untuk mengambil baris lengkap dari heap table

Secara konsep:

```id="0a19ra"
node = root_node
WHILE node IS NOT leaf_node:
    node = navigate(node, 'Jennifer')  -- hanya beberapa langkah
RETURN fetch_from_table(node.ctid)
```

Proses ini jauh lebih cepat karena:

- Tidak perlu membaca seluruh tabel
- Jumlah langkah selalu kecil (logaritmik)

#### Kenapa Table Scan Kadang Masih Dipakai?

Menariknya, table scan tidak selalu buruk. Untuk tabel kecil (misalnya hanya 10–100 baris):

- Overhead untuk menavigasi index bisa lebih besar daripada langsung membaca semua data
- Database query planner bisa memilih table scan karena lebih sederhana dan cepat dalam konteks kecil

Jadi, database biasanya **otomatis memilih strategi terbaik** tergantung kondisi data.

#### Intuisi Sederhana

- **Table scan**: seperti mencari nama di daftar panjang tanpa urutan — kamu harus baca satu per satu
- **Index B-Tree**: seperti menggunakan indeks di buku — langsung lompat ke bagian yang relevan

#### Kesimpulan Praktis

- Gunakan index untuk kolom yang sering dipakai dalam `WHERE`, `JOIN`, atau `ORDER BY`
- Index sangat penting ketika data sudah besar
- Tapi jangan asal membuat index — karena ada biaya tambahan saat `INSERT`, `UPDATE`, dan `DELETE`

Memahami kapan database menggunakan index vs table scan adalah kunci untuk optimasi performa query.

### e. CTID (Pointer ke Tabel)

#### Pengertian Dasar

**CTID** adalah sebuah **pointer (referensi lokasi)** yang disimpan di dalam **leaf node** pada index B-Tree. Pointer ini tidak berisi data lengkap dari sebuah baris, melainkan hanya menunjukkan **di mana baris tersebut berada** di dalam tabel asli.

Dengan kata lain, CTID adalah “alamat” yang dipakai database untuk menemukan data sebenarnya di dalam **heap table** (tabel fisik yang menyimpan seluruh baris).

#### Kenapa CTID Dibutuhkan?

Index dirancang agar ringan dan efisien. Karena itu:

- Index **tidak menyimpan seluruh isi baris**
- Hanya menyimpan **nilai yang di-index + pointer (CTID)**

Tujuannya supaya:

- Struktur index tetap kecil
- Pencarian tetap cepat
- Tidak ada duplikasi data yang tidak perlu

#### Alur Penggunaan CTID

Mari kita lihat alurnya dalam proses pencarian:

1. Database melakukan traversal B-Tree
2. Sampai di leaf node dan menemukan nilai yang dicari (misalnya `"Jennifer"`)
3. Di leaf node tersebut, terdapat CTID
4. Database menggunakan CTID untuk mengambil data lengkap dari heap table

Contoh alur sederhana:

```id="k91c7t"
Leaf Node:
[Jennifer | CTID=(block=5, offset=12)]

→ gunakan CTID
→ ambil baris ke-12 di block ke-5 dari heap table
```

Hasil akhirnya:

- Data lengkap seperti `id`, `name`, `email`, dan kolom lain berhasil diambil

#### Intuisi Sederhana

Bayangkan kamu punya indeks buku:

- Index hanya berisi **kata kunci + nomor halaman**
- Nomor halaman itu adalah CTID
- Untuk membaca isi lengkap, kamu tetap harus membuka halaman tersebut

Jadi:

- B-Tree membantu menemukan “kata kunci”
- CTID membantu menemukan “lokasi isi lengkapnya”

#### Catatan Penting

- CTID bersifat **internal** dan biasanya tidak digunakan langsung oleh developer dalam query sehari-hari
- Nilainya bisa berubah jika baris di-update (karena posisi fisik di disk bisa berubah)
- Meskipun begitu, CTID sangat penting dalam cara kerja internal database untuk menjaga performa tetap optimal

Dengan memahami CTID, kamu bisa melihat bahwa index bukan menyimpan data, melainkan **menjadi peta** yang menunjuk ke lokasi data sebenarnya.

## 3. Hubungan Antar Konsep

Alur kerja B-Tree sebagai index database dapat digambarkan sebagai berikut:

```
Query diterima (WHERE name = 'Jennifer')
        ↓
Traversal dimulai dari Root Node
        ↓
Perbandingan nilai di setiap node (alfabetis/numeris)
        ↓
Bergerak ke Interior Node yang sesuai
        ↓
Mencapai Leaf Node → Nilai ditemukan
        ↓
CTID digunakan untuk mengambil baris dari Heap Table
        ↓
Hasil dikembalikan ke pengguna
```

Ketiga bagian struktur B-Tree (root, interior, leaf) **bekerja bersama** untuk menciptakan jalur pencarian yang efisien. Root node membagi pencarian secara besar-besaran, interior node mempersempit jalur, dan leaf node menyimpan data aktual beserta pointer ke tabel. Tanpa index, proses ini digantikan oleh table scan yang jauh lebih lambat karena harus memeriksa setiap baris secara berurutan.

---

## 4. Kesimpulan

- **B-Tree** adalah jenis index yang paling umum digunakan dalam database relasional.
- Strukturnya terdiri dari **root node**, **interior node**, dan **leaf node** — dengan leaf node menyimpan data terindeks secara berurutan beserta CTID (pointer ke tabel).
- Pencarian dilakukan dengan **traversal dari atas ke bawah**, menggunakan perbandingan nilai untuk memilih jalur yang tepat di setiap level.
- B-Tree sangat efisien untuk query dengan **strict equality** maupun **range**, terutama pada tabel berukuran besar (jutaan hingga miliaran baris).
- Tanpa index, database harus melakukan **table scan** yang lambat. Dengan B-Tree, database cukup melewati beberapa node saja untuk menemukan data yang diinginkan.
- Memahami cara kerja B-Tree membantu membangun **intuisi** tentang kapan sebuah index akan efektif dan kapan penggunaannya mungkin tidak optimal.
