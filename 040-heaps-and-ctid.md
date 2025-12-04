# 📝 Heaps dan CTIDs - Storage Internal PostgreSQL

## 1. Ringkasan Singkat

Video ini menjelaskan mekanisme storage internal PostgreSQL untuk menjawab pertanyaan: "Bagaimana index tahu cara kembali ke table untuk mengambil full row data?" Materi mencakup konsep heap storage (cara PostgreSQL menyimpan data di disk), pages, CTIDs (Tuple IDs), dan bagaimana index menggunakan CTID sebagai pointer untuk kembali ke table. Ini adalah deep dive ke level yang lebih rendah untuk membangun pemahaman fundamental tentang cara kerja indexes.

## 2. Konsep Utama

### a. Review: Pointer dari Index ke Table

Pada video sebelumnya, kita melihat bagaimana database menggunakan **index** untuk mempercepat pencarian data. Contoh sederhana yang sering dipakai adalah query berikut:

```sql
SELECT * FROM users WHERE last_name = 'Smith';
```

Untuk menjawab query ini, database melakukan beberapa langkah penting:

1. **Menelusuri index** terlebih dahulu.
   Misalnya ada index bernama `idx_last_name`. Database akan mencari entri `'Smith'` di dalam index ini.

2. **Index hanya menyimpan kolom yang di-index** — dalam hal ini hanya `last_name`.
   Namun query meminta `SELECT *`, yang berarti seluruh kolom harus diambil.

3. **Database harus kembali ke table utama** untuk mengambil seluruh kolom.
   Di sinilah konsep _pointer_ muncul sebagai kunci.

#### Kenapa Butuh Pointer?

Satu hal yang penting dipahami: **index bukan bagian dari table**, melainkan struktur data yang terpisah. Index disusun khusus untuk mempercepat pencarian, tetapi tidak menyimpan semua data table.

Tanpa mekanisme tambahan, ada masalah besar:

- Setelah index menemukan baris yang cocok, database **tidak otomatis tahu di mana letak baris lengkapnya di table**.
- Jika tidak ada cara untuk melompat langsung ke baris itu, database harus **scan seluruh table** lagi untuk mencari data lengkapnya — dan ini jelas sangat lambat.

Karena itu, index membutuhkan **pointer** — semacam "alamat" atau "koordinat" — yang mengarah langsung ke lokasi row di table. Dengan pointer:

- Setelah index menemukan `'Smith'`, database dapat **langsung melompat** ke baris yang tepat.
- Tidak perlu membaca baris table lain.
- Fetch data menjadi jauh lebih cepat.

#### Ilustrasi Cara Kerja Pointer

Bayangkan index sebagai daftar nama yang rapi dan terurut:

| last_name | pointer ke row |
| --------- | -------------- |
| Adams     | → row #12      |
| Brown     | → row #87      |
| Smith     | → row #203     |
| Smith     | → row #204     |

Saat index menemukan `'Smith'`, ia melihat pointer yang menyertai entri tersebut. Pointer inilah yang mengarah langsung ke lokasi baris di table:

```
Index:
  'Smith' → pointer → Row 203 di table users
                    → Row 204 di table users

Table:
  Row 203 → { first_name: 'John', last_name: 'Smith', email: '...' }
  Row 204 → { first_name: 'Alice', last_name: 'Smith', email: '...' }
```

Database kemudian mengambil data lengkap dari table sesuai kebutuhan query.

#### Pertanyaan Utama dari Video

> “What is that connection between the separate data structure and the rest of the data that we likely need?”

Jawabannya adalah **pointer dari index ke table**.
Pointer inilah yang menjadi jembatan antara dua struktur data yang terpisah: index dan table utama.

Tanpa pointer, index tidak banyak berguna untuk query seperti `SELECT *`.
Dengan pointer, index benar-benar mempercepat kinerja database.

### b. Heap Storage – Cara PostgreSQL Menyimpan Data

#### Definisi Heap

Dalam PostgreSQL, **heap** adalah struktur penyimpanan dasar tempat baris-baris data (rows) disimpan. Nama _heap_ cukup menggambarkan cara kerjanya: mirip seperti “tumpukan” atau “kumpulan barang” yang diletakkan di mana saja ada ruang kosong. Tidak ada aturan bahwa data harus disimpan berurutan, tidak perlu disortir, dan tidak harus mengikuti pola tertentu.

Secara sederhana, PostgreSQL hanya melakukan:

```
Heap = "Letakkan row di mana saja ada space kosong"
```

Itu berarti sebuah row bisa berada di page 0, page 12, page 300, atau page mana saja — selama masih tersedia ruang.

#### Karakteristik Heap

Karena tidak ada organizing principle khusus, hasilnya kira-kira seperti berikut:

- Row bisa berada di **page 0**
- Row berikutnya mungkin berada di **page 60**
- Row lain mungkin berada di **page 432**
- PostgreSQL hanya memastikan ada **space kosong**, tanpa aturan urutan apa pun

Struktur ini memberi PostgreSQL fleksibilitas tinggi dalam operasi penulisan data.

#### Mengapa PostgreSQL Menggunakan Heap?

Penggunaan heap memberikan beberapa keuntungan besar, terutama untuk beban kerja yang sering melakukan INSERT.

| Aspek                  | Benefit                                                                    |
| ---------------------- | -------------------------------------------------------------------------- |
| **INSERT performance** | Sangat cepat — tidak perlu menggeser atau menata ulang data yang sudah ada |
| **Write efficiency**   | Cukup temukan ruang kosong terdekat dan tulis di situ                      |
| **Simplicity**         | Tidak perlu menjaga data dalam kondisi terurut                             |

Dengan kata lain, heap memprioritaskan kecepatan tulis dan kemudahan manajemen data.

#### Visualisasi Cara Kerja Heap

Anda bisa membayangkan database PostgreSQL memiliki banyak “halaman” penyimpanan (pages). Masing-masing page dapat berisi beberapa row. Karena heap tidak memiliki aturan pengurutan, posisi row sangat bergantung pada di mana space kosong tersedia.

```
┌────────────────────────────────────────────┐
│           PostgreSQL Database              │
├────────────────────────────────────────────┤
│  Page 0  │  Page 1  │  Page 2  │  Page 3   │
├──────────┼──────────┼──────────┼───────────┤
│ Row 1    │ Row 7    │          │ Row 15    │
│ Row 2    │ Row 8    │ (empty)  │ Row 16    │
│ Row 3    │          │          │           │
│          │ (empty)  │          │ (empty)   │
└──────────┴──────────┴──────────┴───────────┘
```

Pada ilustrasi di atas, terlihat bahwa:

- Page 0 punya beberapa row
- Page 1 punya beberapa row dan dua slot kosong
- Page 2 kosong
- Page 3 punya beberapa row lagi

Tidak ada hubungan antara row di page satu dengan row di page lainnya — ini murni “asal ada space”.

#### Contoh Proses INSERT

Mari kita lihat bagaimana PostgreSQL menambahkan row baru ke dalam heap.

```sql
INSERT INTO users VALUES ('Alice', 'Smith', ...);
```

Proses internalnya kira-kira seperti ini:

1. PostgreSQL bertanya, **“Page mana yang masih punya space?”**
2. Ia melakukan scan ringan terhadap pages:

   - Page 0: penuh
   - Page 1: ternyata masih ada space!

3. PostgreSQL langsung **menulis row baru** pada slot kosong di Page 1 (misalnya pada posisi ke-9).
4. Proses selesai — dan semuanya berlangsung sangat cepat.

Karena PostgreSQL tidak perlu:

- menata ulang row existing,
- menjaga urutan tertentu,
- atau memindahkan block data lain,

maka operasi INSERT dapat dieksekusi secara efisien.

### c. Pages - Unit Storage di PostgreSQL

#### Apa itu Page?

Dalam PostgreSQL, **page** adalah unit penyimpanan paling dasar. Anda bisa membayangkannya sebagai “blok fisik” berukuran tetap yang digunakan untuk menyimpan semua data—baik itu **tables**, **indexes**, maupun struktur internal lainnya. Setiap page memiliki ukuran yang sama, dan secara default PostgreSQL menggunakan ukuran **8KB per page**.

Satu database pada akhirnya hanyalah kumpulan page yang saling berdampingan secara berurutan. PostgreSQL membaca dan menulis data dalam satuan page, bukan dalam ukuran baris atau kolom.

#### Struktur Pages dalam Database

Representasi sederhananya dapat digambarkan seperti ini:

```
Database = Collection of pages

┌──────────────────────┐
│      Page 0          │  8KB
├──────────────────────┤
│      Page 1          │  8KB
├──────────────────────┤
│      Page 2          │  8KB
├──────────────────────┤
│      Page 3          │  8KB
├──────────────────────┤
│      ...             │
├──────────────────────┤
│      Page 999        │  8KB
└──────────────────────┘
```

Setiap page diperlakukan sebagai unit yang utuh. Ketika PostgreSQL ingin membaca row yang berada di page 120, misalnya, ia akan memuat **seluruh page 120** ke memori, bukan hanya row tertentu saja. Inilah alasan mengapa konsep page sangat penting dalam performa I/O database.

#### Apa yang Ada di Dalam Satu Page?

Di dalam sebuah page, PostgreSQL menyimpan beberapa row sekaligus. Setiap row ditempatkan di **position** atau **offset** tertentu di dalam page. Offset ini di-number mulai dari 0, sama seperti page itu sendiri.

Contohnya:

```
┌────────────────────────────────────┐
│         Page 5 (8KB)               │
├────────────────────────────────────┤
│ Position 0: Row data (120 bytes)   │
│ Position 1: Row data (200 bytes)   │
│ Position 2: Row data (150 bytes)   │
│ Position 3: Row data (180 bytes)   │
│ ...                                 │
│ Position 15: Row data (95 bytes)   │
│ (remaining space available)         │
└────────────────────────────────────┘
```

Di sini terlihat:

- Page 5 berukuran tetap **8KB**
- Baris-baris data ditempatkan pada posisi tertentu dalam page
- Sisa ruang pada page dapat digunakan untuk row baru (selama masih cukup)

Page memiliki struktur internal lain seperti header page, unused space, dan line pointer array. Namun konsep di atas sudah cukup memberi gambaran bagaimana row diposisikan secara fisik di dalam satu page.

#### Key Points yang Perlu Diperhatikan

- Pages selalu di-number dari 0:
  `Page 0, Page 1, Page 2, ...`
- Setiap row di dalam page juga punya nomor posisi (offset) mulai dari 0.
- Semua data—baik table rows maupun index nodes—disimpan dalam bentuk page.
- Operasi disk PostgreSQL selalu bekerja dalam satuan page, sehingga layout ini sangat memengaruhi performa query.

### d. CTID (Tuple ID) - Physical Row Identifier

#### Definisi CTID

Dalam PostgreSQL, **CTID** adalah identitas fisik dari sebuah row. Jika sebelumnya kita membahas bahwa data disimpan dalam **pages**, dan setiap page memiliki banyak **positions** (offset), maka CTID adalah pasangan informasi yang menunjukkan lokasi persis sebuah row berada di disk.

Format CTID adalah:

```
(page_number, position_within_page)
```

Contoh:

- `(0,1)` berarti row tersebut berada di **Page 0**, **Position 1**
- `(12,4)` berarti row berada di **Page 12**, **Position 4**

Dengan kata lain, CTID adalah “alamat rumah” sebuah row di dalam storage fisik PostgreSQL.

#### CTID sebagai Hidden System Column

Meskipun CTID dimiliki oleh setiap row, kolom ini **tidak muncul** ketika Anda melakukan `SELECT *`. PostgreSQL menyembunyikannya secara default. Namun CTID tetap ada dan dapat kita lihat jika kita memintanya secara eksplisit.

Perhatikan contoh berikut:

```sql
-- SELECT * tidak menampilkan CTID
SELECT * FROM reservations;
-- id | room_id | reservation_period | booking_status
-- ----+---------+--------------------+---------------
--  1 |   101   | [...]              | confirmed
```

Ketika kita meminta CTID secara eksplisit:

```sql
SELECT ctid, * FROM reservations;
-- ctid   | id | room_id | reservation_period | booking_status
-- -------+----+---------+--------------------+---------------
-- (0,1)  | 1  | 101     | [...]              | confirmed
-- (0,2)  | 2  | 102     | [...]              | confirmed
-- (0,3)  | 3  | 101     | [...]              | canceled
```

Sekarang kita melihat CTID disajikan sebagai pasangan `(page, position)`.

#### Penjelasan Hasil Query

Kita bisa membaca CTID seperti berikut:

```
(0,1) → Page 0, Position 1 → Di sinilah row #1 disimpan
(0,2) → Page 0, Position 2 → Di sinilah row #2 disimpan
(0,3) → Page 0, Position 3 → Di sinilah row #3 disimpan
```

Artinya, PostgreSQL menyimpan ketiga row tersebut pada page yang sama (page 0) tetapi pada posisi yang berbeda di dalam page tersebut.

#### CTID Memberikan Physical Disk Location

Jika kita visualisasikan isi page di dalam database, kira-kira tampilannya seperti ini:

```
Database di disk:

Page 0:
├─ Position 1: [1, 101, '[2024-09-01...]', 'confirmed']   ← Row dengan ctid (0,1)
├─ Position 2: [2, 102, '[2024-09-02...]', 'confirmed']   ← Row dengan ctid (0,2)
└─ Position 3: [3, 101, '[2024-09-03...]', 'canceled']    ← Row dengan ctid (0,3)
```

Dengan mengetahui CTID, PostgreSQL dapat **melompat langsung** ke lokasi fisik row tersebut. Tidak perlu scan tabel, tidak perlu mencari berdasarkan nilai kolom lain — cukup ambil `(page_number, position)` dan ambil row dari alamat itu. Inilah yang membuat CTID sangat penting dalam mekanisme internal PostgreSQL, khususnya untuk operasi seperti `UPDATE`, `DELETE`, atau saat index menunjuk langsung ke lokasi row di disk.

### e. Querying by CTID (dan Mengapa TIDAK Boleh Dilakukan)

#### Query dengan CTID: Bisa, Tapi Tidak Dianjurkan

Secara teknis, PostgreSQL **memungkinkan** kita melakukan query berdasarkan CTID. Karena CTID menunjukkan lokasi fisik sebuah row, PostgreSQL bisa langsung melompat ke page dan position tersebut tanpa harus melakukan scan atau traversal index.

Contoh:

```sql
-- Ambil row di Page 0, Position 2
SELECT * FROM reservations WHERE ctid = '(0,2)';
-- ✅ Technically works

-- Result:
-- id | room_id | reservation_period | booking_status
-- ---+---------+--------------------+---------------
--  2 |   102   | [...]              | confirmed
```

Secara performa, ini sangat cepat. Namun meskipun bisa, **tidak berarti aman digunakan**.

#### Mengapa Tidak Boleh Query Berdasarkan CTID?

Ada dua alasan utama: **CTID bersifat volatile** dan **dapat berubah kapan saja**.

#### Alasan 1 — CTID Bersifat Volatile (Berubah setelah UPDATE)

CTID adalah alamat fisik row, bukan identitas logis. Setelah sebuah row diubah—apalagi jika ukuran row menjadi lebih besar—PostgreSQL mungkin perlu memindahkan row tersebut ke page lain yang memiliki ruang cukup.

Contoh:

```sql
-- Cek lokasi awal row id = 1
SELECT ctid, * FROM users WHERE id = 1;
-- (0,2) | 1 | 'Alice'

-- UPDATE membuat row lebih besar
UPDATE users SET bio = 'Very long text...' WHERE id = 1;

-- CTID berubah
SELECT ctid, * FROM users WHERE id = 1;
-- (84,5) | 1 | 'Alice'   ← Bergeser ke Page 84!
```

Penjelasan:

1. Row awal berada di `(0,2)`
2. Setelah `UPDATE`, ukuran row bertambah
3. Page 0 mungkin sudah padat dan tidak bisa menampung row yang lebih besar
4. PostgreSQL memindahkan row ke page yang lebih longgar, misalnya Page 84
5. CTID berubah menjadi `(84,5)`

Jika pada awalnya Anda mengandalkan CTID `(0,2)`, maka identifier itu sudah tidak berlaku lagi.

#### Alasan 2 — VACUUM Dapat Menyusun Ulang Row

VACUUM bertugas membersihkan dead tuples dan mengompaktkan ruang dalam pages. Hal ini dapat menggeser row ke position lain atau memindahkannya antar page. Hasilnya, CTID dapat berubah meskipun Anda tidak melakukan UPDATE.

Contoh:

```sql
-- Sebelum VACUUM
SELECT ctid, id FROM products;
-- (0,1) | 1
-- (0,2) | 2  ← Akan dihapus
-- (0,3) | 3
-- (5,8) | 4

DELETE FROM products WHERE id = 2;

-- Setelah VACUUM
SELECT ctid, id FROM products;
-- (0,1) | 1
-- (0,2) | 3  ← Dulu (0,3)
-- (0,3) | 4  ← Dulu (5,8)
```

VACUUM mengompaktkan page untuk mengisi celah kosong, sehingga posisi row berubah. CTID yang lama tidak dapat dipakai lagi.

#### Mengapa CTID Tidak Cocok sebagai Identifier?

CTID memang unik pada saat tertentu, namun:

- Tidak stabil
- Berubah saat UPDATE
- Berubah saat VACUUM
- Bukan penentu identitas logis row
- Sama sekali tidak cocok untuk referensi jangka panjang
- Tidak dapat dipakai sebagai foreign key, constraint, atau lookup yang aman

Bandingkan dengan primary key:

| Karakteristik              | CTID                             | Primary Key  |
| -------------------------- | -------------------------------- | ------------ |
| **Stable**                 | ❌ Berubah setelah UPDATE/VACUUM | ✅ Tetap     |
| **Deterministic**          | ❌ Tidak dapat diprediksi        | ✅ Konsisten |
| **Volatile**               | ✅ Sangat volatile               | ❌ Tidak     |
| **Suitable as identifier** | ❌ Sama sekali tidak             | ✅ Ya        |

#### Peringatan dari Instruktur

> “Don’t do that. These CTIDs can and will change. They’re not primary keys, they’re not stable, they’re not deterministic, they’re volatile.”

Dengan kata lain, CTID memang berguna untuk memahami bagaimana PostgreSQL bekerja secara internal—terutama saat mempelajari storage dan indexing. Namun untuk aplikasi nyata, **selalu gunakan primary key**, bukan CTID.

### f. CTID sebagai Pointer dalam Index - The Missing Link!

#### Apa yang Dimaksud dengan "Pointer" dalam Index?

Pada video sebelumnya, muncul pertanyaan penting:

```
"Every index contains a pointer back to the table"
```

Bagian ini akhirnya terjawab sekarang:

```
"Every index contains the CTID"
```

Artinya, pointer yang dimaksud bukanlah alamat memori seperti di bahasa pemrograman, tetapi sebuah nilai khusus yang dikenal sebagai **CTID**.

CTID inilah yang memberi tahu PostgreSQL di mana tepatnya sebuah row disimpan di dalam table.

#### CTID = Pointer Sesungguhnya

Di dalam index, setiap entry selalu berpasangan antara nilai yang diindeks dan CTID. Bentuknya kira-kira seperti ini:

```
Index Entry:
┌─────────────────────────────┐
│ Indexed value | CTID        │
├───────────────┼─────────────┤
│ 'Smith'       | (0,10)      │  ← CTID = pointer!
│ 'Johnson'     | (2,5)       │
│ 'Williams'    | (5,8)       │
└───────────────┴─────────────┘
```

Format CTID berupa `(page, position)`.
Misalnya `(0,10)` berarti:

- page (halaman) ke-0 di heap/table
- posisi row ke-10 pada halaman tersebut

Jadi CTID adalah **koordinat** untuk langsung melompat ke row tertentu tanpa perlu melakukan scanning.

#### Alur Lengkap: Bagaimana Index Lookup Bekerja dengan CTID

Misalkan kita menjalankan query:

```sql
SELECT * FROM users WHERE last_name = 'Smith';
```

Mari kita uraikan prosesnya langkah demi langkah.

##### 1. PostgreSQL Menelusuri Index

Database akan mulai mencari nilai `'Smith'` di index `idx_last_name`. Tergantung jenis index, pencarian ini dilakukan dengan:

- binary search
- atau tree traversal (misalnya pada B-tree)

Setelah ketemu, kita mendapatkan pasangan berikut:

```
'Smith' → CTID: (0,10)
```

##### 2. PostgreSQL Menggunakan CTID Sebagai Pointer

CTID menunjukkan lokasi persis row tersebut.

```
"Go to Page 0, Position 10"
```

Ini adalah lompatan langsung (**direct jump**), tanpa scanning sama sekali.

##### 3. PostgreSQL Mengambil Row Lengkap dari Table (Heap)

Setelah tiba di page yang dimaksud, database membaca isi row:

```
[id: 123, first: 'John', last: 'Smith', email: 'john@...']
```

Row inilah yang kemudian dikembalikan sebagai hasil query.

#### Visualisasi Alur Lookup Secara Utuh

```
┌──────────────────┐         ┌─────────────────────────┐
│  Index           │         │  Table (Heap)           │
│  (idx_last_name) │         │                         │
├──────────────────┤         │  Page 0:                │
│ 'Johnson' (2,5)  │         │  ├─ Pos 10: [Smith...]  │◄─┐
│ 'Smith'   (0,10) │─────────┼──┘                      │  │
│ 'Williams'(5,8)  │         │  Page 2:                │  │
└──────────────────┘         │  ├─ Pos 5: [Johnson...] │  │
        ↓                    │                         │  │
   1. Find 'Smith'           │  Page 5:                │  │
   2. Get CTID: (0,10)       │  └─ Pos 8: [Williams..] │  │
   3. Jump to table ──────────────────────────────────────┘
```

Diagram di atas menggambarkan hubungan antara struktur index dan lokasi row di dalam table. Setiap langkah lookup menjadi sangat efisien karena CTID menyediakan koordinat yang tepat.

#### Mengapa CTID Membuat Proses Lookup Sangat Efisien?

Tanpa CTID (secara teoretis):

```
Query  → Index → "I found 'Smith' somewhere"
       → Harus scan seluruh table untuk menemukannya
       → Kompleksitas: O(n)
```

Dengan CTID (realita di PostgreSQL):

```
Query  → Index → "I found 'Smith' at (0,10)"
       → Langsung lompat ke Page 0, Position 10
       → Kompleksitas: O(1)
```

Inilah inti performa index: bukan hanya menemukan nilai yang dicari, tetapi menyediakan **pointer langsung** menuju lokasi row di table.

### g. Key Insight: Index Storage vs Table Storage

#### Apa yang Disimpan oleh Index

Index bukanlah salinan lengkap dari table. Struktur ini hanya menyimpan informasi yang diperlukan untuk melakukan pencarian cepat. Secara khusus, isi index terdiri dari dua hal penting:

```
1. Nilai dari kolom yang di-index
2. CTID (pointer menuju lokasi row di table)
```

Sebagai contoh, untuk index `idx_last_name`:

```
┌──────────┬────────┐
│ last_name│ CTID   │
├──────────┼────────┤
│ 'Doe'    │ (1,3)  │
│ 'Smith'  │ (0,10) │
└──────────┴────────┘
```

Di sini terlihat jelas bahwa index hanya menyimpan **nilai kolom yang diindeks** dan **CTID** sebagai penunjuk lokasi fisik row yang sebenarnya ada di table. Tidak ada kolom lain, tidak ada data tambahan.

#### Apa yang Disimpan oleh Table (Heap Storage)

Table, atau lebih tepatnya _heap_, adalah tempat penyimpanan data sebenarnya. Semua kolom dan seluruh isi row disimpan secara lengkap pada lokasi fisik tertentu, yaitu halaman (page) dan posisi (offset) di dalam halaman tersebut.

Contohnya, CTID `(0,10)` berarti baris tersebut berada di:

```
Page 0, Position 10:
[id: 123, first_name: 'John', last_name: 'Smith',
 email: 'john@example.com', created_at: '2024-01-01', ...]
```

Index hanya memberi tahu **di mana** datanya berada. Table-lah yang menyimpan **apa** datanya.

#### Pembagian Peran: Kenapa Harus Dipisah?

Kedua komponen ini bekerja sama, tetapi masing-masing memiliki tanggung jawab yang berbeda agar sistem tetap cepat dan efisien.

| Component | Responsibility                                                  | Optimization                                                                           |
| --------- | --------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **Index** | Melakukan pencarian cepat berdasarkan nilai kolom yang diindeks | Memanfaatkan struktur B-tree dan urutan data untuk traversal cepat                     |
| **CTID**  | Menjadi jembatan antara index dan table                         | Memberikan alamat fisik yang tepat, sehingga PostgreSQL dapat melompat langsung ke row |
| **Heap**  | Menyimpan data lengkap dari setiap row                          | Optimized untuk operasi tulis (insert/update) serta penyimpanan yang fleksibel         |

Pembagian tugas ini membuat query dapat berjalan cepat: index menemukan nilai dengan efisien, CTID memberikan lokasi tepat, dan heap menyediakan data lengkapnya.

### h. Implications untuk Index Design

#### Pemahaman Dasar yang Sekarang Kita Miliki

Setelah memahami bahwa setiap index menyimpan pasangan **nilai yang diindeks** dan **CTID**, kita bisa mulai melihat konsekuensi desainnya. Informasi ini sangat penting karena menentukan ukuran index, performa query, serta strategi indexing yang tepat.

#### 1. Index Selalu Menyimpan CTID

Setiap entry dalam index wajib memiliki CTID sebagai pointer ke lokasi fisik row di tabel. Artinya, berapa pun ukuran nilai yang diindeks, selalu ada tambahan 6 byte untuk CTID.

```
Index size = (indexed data) + (CTID per entry)

Ukuran CTID:
- 4 bytes untuk page number
- 2 bytes untuk position dalam page
```

Dengan kata lain, semakin banyak baris dan semakin besar nilai yang diindeks, semakin besar pula index yang terbentuk.

#### 2. Overhead yang Harus Diperhitungkan

Karena setiap entry memiliki tambahan 6 byte, total ukuran index bisa dihitung secara sederhana sebagai berikut:

```
Every index entry = indexed value + 6 bytes CTID
```

Contoh hitungan:

- Jika Anda mengindeks kolom **INTEGER (4 bytes)**:

  ```
  Total per entry = 4 + 6 = 10 bytes
  ```

- Jika Anda mengindeks kolom **TEXT dengan panjang rata-rata 20 bytes**:

  ```
  Total per entry = 20 + 6 = 26 bytes
  ```

Konsekuensinya, index pada kolom TEXT biasanya jauh lebih besar dibanding index pada kolom berbasis angka. Ini penting ketika Anda merancang index pada tabel besar, karena ukuran index berpengaruh langsung pada performa dan penggunaan memori.

#### 3. Covering Indexes (Teaser untuk Pembahasan Berikutnya)

Untuk memahami desain index yang lebih canggih, kita perlu mengenal konsep **covering index**.

```
Normal index: hanya menyimpan indexed value + CTID

Covering index: menyimpan indexed value + kolom tambahan + CTID
```

Manfaatnya sangat signifikan:

- PostgreSQL dapat memenuhi hasil query langsung dari index
- Tidak perlu melakukan table lookup ke heap
- Mengurangi random I/O dan mempercepat query secara drastis

Contoh intuitifnya:
Jika sebuah query hanya membutuhkan kolom yang sudah tersedia dalam index, PostgreSQL dapat memprosesnya tanpa membaca table sama sekali.

Konsep ini akan dibahas lebih mendalam pada materi berikutnya, tetapi sudah cukup untuk memahami bahwa desain index bukan hanya soal “apa yang sering di-query”, melainkan juga soal “apakah query dapat dipenuhi langsung dari index”.

## 3. Hubungan Antar Konsep

### Alur Pemahaman Complete Picture:

```
1. User Query
   ↓
   SELECT * FROM users WHERE last_name = 'Smith'

2. Query Planner
   ↓
   "Use index idx_last_name"

3. Index Traversal (B-tree, covered later)
   ↓
   Navigate tree structure
   ↓
   Find: 'Smith' → Associated CTID: (0,10)

4. CTID Lookup
   ↓
   "Go to Heap: Page 0, Position 10"

5. Heap Access
   ↓
   Read Page 0 from disk (or cache)
   ↓
   Get Position 10 within that page
   ↓
   Full row data retrieved

6. Return Result
   ↓
   [id: 123, first_name: 'John', last_name: 'Smith', ...]
```

### Hierarchy of Storage Concepts:

```
PostgreSQL Database
│
├─ Heap Storage (Table data)
│  ├─ Pages (8KB blocks)
│  │  ├─ Page 0
│  │  │  ├─ Position 0 → Row
│  │  │  ├─ Position 1 → Row
│  │  │  └─ Position n → Row
│  │  ├─ Page 1
│  │  │  └─ ...
│  │  └─ Page n
│  │
│  └─ CTID system column
│     └─ (page_number, position) → Physical location
│
└─ Index Storage (Separate structure)
   ├─ B-tree (or other structure)
   ├─ Indexed values (sorted/organized)
   └─ CTIDs (pointers to heap)
```

### Mengapa Heap + CTID Design Masuk Akal:

```
Design Goals:
├─ Fast writes (inserts)
│  └─ Solution: Heap (no sorting, write anywhere)
│
├─ Fast reads (selects with WHERE clause)
│  └─ Solution: Index (organized structure)
│
└─ Fast connection between index & table
   └─ Solution: CTID (direct physical address)

Result: Optimal balance for OLTP workloads
```

### Trade-offs dari Design Ini:

**Pros:**

```
✅ INSERT performance excellent (heap = write anywhere)
✅ Index lookups very fast (B-tree + direct CTID access)
✅ Flexible schema (easy to add/remove indexes)
```

**Cons:**

```
❌ Full table scans slow (no inherent order in heap)
   → Solution: Use indexes!

❌ Index maintenance cost on writes (must update CTIDs)
   → Solution: Only index what you need

❌ VACUUM needed to reclaim space (deleted rows leave gaps)
   → Solution: Regular maintenance (autovacuum)
```

### Mental Model: Analogi Library

```
HEAP = Gudang buku (storage room)
       ├─ Shelves = Pages
       ├─ Positions on shelf = Row positions
       └─ Books arranged by acquisition order (not sorted)

INDEX = Catalog system
        ├─ Organized alphabetically/by topic
        ├─ Entry: "Book Title" → "Shelf 5, Position 3"
        └─ CTID = "Shelf 5, Position 3" (location code)

Lookup process:
1. Check catalog (index) → "Pride and Prejudice"
2. Get location: Shelf 5, Position 3 (CTID)
3. Walk to Shelf 5 (page)
4. Grab book at Position 3 (row)
5. Done! No need to search entire warehouse.
```

### Progression of Understanding:

```
Video 1 (Introduction to Indexes):
"Index contains a pointer to table"
        ↓
        What is this pointer? 🤔

Video 2 (This video):
"Pointer = CTID = (page, position)"
        ↓
        Aha! Now I understand the mechanism!

Next videos:
"How does index organize data for fast lookup?"
        ↓
        B-tree structure, etc.
```

## 4. Kesimpulan

**Key Takeaways:**

1. **Heap Storage:**

   - PostgreSQL menggunakan heap storage untuk table data
   - Heap = "pile" - rows ditempatkan di mana pun ada space
   - Benefit: INSERT sangat cepat (tidak perlu maintain order)

2. **Pages:**

   - Unit storage dasar di PostgreSQL
   - Default size: 8KB per page
   - Numbered mulai dari 0: Page 0, Page 1, Page 2, ...
   - Di dalam page ada positions untuk rows

3. **CTID (Tuple ID):**

   - Physical location identifier: `(page_number, position)`
   - Hidden system column (tidak terlihat di SELECT \*)
   - Format: `(0,10)` = Page 0, Position 10

4. **⚠️ JANGAN gunakan CTID sebagai identifier:**

   - ❌ Volatile: Berubah saat UPDATE
   - ❌ Not stable: Berubah saat VACUUM
   - ❌ Not deterministic: Unpredictable
   - ✅ Gunakan PRIMARY KEY untuk identifikasi!

5. **CTID adalah "Pointer" dalam Index:**

   ```
   Index Entry = (indexed_value, CTID)

   Example:
   'Smith' → (0,10)

   Meaning: "Value 'Smith' is located at Page 0, Position 10"
   ```

6. **Complete Lookup Flow:**

   ```
   Query → Index (find value, get CTID)
        → Heap (jump to CTID location)
        → Return full row
   ```

7. **Efficiency:**
   - Tanpa CTID: Must scan table (slow)
   - Dengan CTID: Direct jump O(1) (fast!)

**Prinsip emas:**

> "Every index contains the CTID, because that is the actual pointer that gets you back to the table to find the rest of your data"

**Yang sudah kita pahami sekarang:**

- ✅ Index adalah separate structure (Video 1)
- ✅ Index contains copy of data (Video 1)
- ✅ Index contains pointer = **CTID** (Video 2) ← New!
- ✅ Heap storage mechanism (Video 2) ← New!

**Yang akan dipelajari selanjutnya:**

- Bagaimana index organize data untuk fast traversal? (B-tree structure)
- Index types dan kapan menggunakan apa
- Practical index creation strategies
- Performance tuning dengan indexes

Dengan memahami heap, pages, dan CTIDs, kita sekarang punya complete picture tentang bagaimana index benar-benar bekerja di level internal PostgreSQL! 🎯
