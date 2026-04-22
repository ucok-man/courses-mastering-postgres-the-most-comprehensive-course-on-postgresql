# 📝 Tipe Data untuk Primary Key di Database

## 1. Ringkasan Singkat

Video ini membahas rekomendasi praktis tentang tipe data apa yang sebaiknya digunakan untuk **primary key** di database relasional. Pembicara (Aaron) membagi topik menjadi dua kategori utama: **Integer** (khususnya `BIGINT`) dan **UUID/ULID**. Ia menjelaskan kelebihan dan kekurangan masing-masing, kapan harus menggunakan salah satunya, serta bagaimana menangani masalah keamanan yang sering dikaitkan dengan primary key berbasis integer.

---

## 2. Konsep Utama

### a. Integer sebagai Primary Key (Rekomendasi Utama)

Dalam praktik database modern, penggunaan integer sebagai primary key bisa dibilang adalah pilihan default yang paling aman dan efisien. Bahkan, untuk sekitar 98% kasus penggunaan, pendekatan ini sudah lebih dari cukup.

Namun, ada satu detail penting yang sering terlewat: bukan sekadar menggunakan integer, tetapi memilih tipe integer yang tepat—yaitu `BIGINT`, bukan `INT`.

#### Mengapa `BIGINT` lebih direkomendasikan daripada `INT`?

Secara umum, kita memang diajarkan untuk memilih tipe data sekecil mungkin agar hemat storage. Tapi untuk primary key, aturan ini sengaja “dilanggar” demi menghindari masalah besar di masa depan.

Mari kita lihat perbedaannya:

- `INT` (32-bit) memiliki batas maksimum sekitar 2,1 miliar nilai.
- `BIGINT` (64-bit) mampu menampung hingga sekitar 9,2 kuintilion nilai.

Sekilas, angka 2,1 miliar terdengar sangat besar. Tapi dalam sistem yang berkembang pesat—misalnya aplikasi dengan jutaan user aktif atau data transaksi yang terus bertambah—batas ini bisa tercapai lebih cepat dari yang dibayangkan.

Ada contoh nyata dari hal ini: Basecamp, sebuah platform manajemen proyek, pernah mengalami kegagalan sistem karena kehabisan ruang pada kolom primary key bertipe integer. Artinya, mereka tidak bisa lagi menambahkan data baru karena semua kemungkinan nilai sudah habis.

Dengan `BIGINT`, risiko ini praktis tidak ada. Kapasitasnya настолько besar sehingga dalam konteks aplikasi modern, bisa dianggap “tak terbatas”.

#### Cara penulisan yang direkomendasikan

Untuk membuat primary key dengan `BIGINT` yang auto-increment (bertambah otomatis), kita bisa menggunakan fitur identity:

```sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    -- kolom lainnya...
);
```

Penjelasannya:

- `BIGINT`: menentukan tipe data dengan kapasitas besar.
- `GENERATED ALWAYS AS IDENTITY`: membuat nilai `id` otomatis bertambah setiap kali ada data baru.
- `PRIMARY KEY`: menjadikan kolom ini sebagai identitas unik setiap baris.

Dengan ini, kita tidak perlu lagi mengatur nilai ID secara manual—database akan menanganinya dengan aman dan konsisten.

#### Kenapa integer auto-increment sangat efisien dengan B-tree?

Sebagian besar database (seperti PostgreSQL, MySQL, dll.) menggunakan struktur indeks bernama B-tree untuk mengelola data agar cepat dicari dan diakses.

Keuntungan besar dari integer auto-increment adalah sifatnya yang selalu bertambah secara berurutan (monotonically increasing). Artinya:

- Data baru selalu memiliki nilai yang lebih besar dari sebelumnya.
- Insert data akan selalu terjadi di “ujung” struktur B-tree (biasanya sisi kanan).

Bayangkan struktur seperti ini:

```
        [100]
       /     \
   [50]      [150]  ← data baru masuk ke sini
  /    \    /    \
[25]  [75][125] [175] → [200] → [201] ...
```

Karena penambahan selalu terjadi di satu arah:

- Struktur B-tree tetap stabil dan seimbang.
- Minim kebutuhan rebalancing (penyusunan ulang node).
- Performa insert menjadi sangat cepat dan konsisten.

Bandingkan dengan primary key acak (misalnya UUID), di mana data bisa masuk ke bagian mana saja dalam tree. Hal ini sering memicu rebalancing yang lebih sering, sehingga performa bisa menurun.

#### Ringkasan pemahaman

Menggunakan `BIGINT` sebagai primary key bukan hanya soal kapasitas besar, tetapi juga soal desain yang tahan terhadap pertumbuhan sistem.

Kombinasi antara:

- kapasitas hampir tak terbatas,
- auto-increment yang sederhana,
- dan kompatibilitas optimal dengan struktur B-tree,

menjadikannya pilihan yang sangat solid untuk sebagian besar aplikasi.

### b. UUID sebagai Primary Key

Selain integer, alternatif lain yang cukup populer untuk primary key adalah UUID (Universally Unique Identifier). UUID merupakan nilai 128-bit yang dirancang untuk menjadi unik secara global, bahkan tanpa perlu koordinasi langsung dengan database.

Artinya, kita bisa membuat ID di sisi aplikasi (misalnya di backend service) tanpa harus “bertanya” ke database terlebih dahulu—ini sangat berguna dalam sistem terdistribusi.

Saat ini, terdapat 7 versi UUID yang berbeda (seperti v1, v4, v7, dll.), masing-masing dengan karakteristik sendiri. Selain itu, ada juga ULID (Universally Unique Lexicographically Sortable Identifier) yang mencoba menggabungkan keunikan UUID dengan kemampuan untuk diurutkan.

#### Dua masalah utama UUID (terutama UUID acak)

Meskipun fleksibel, UUID—khususnya yang bersifat acak seperti UUID v4—memiliki beberapa kekurangan yang perlu dipahami dengan baik.

#### 1. Ukuran lebih besar

Dibandingkan dengan integer:

- `BIGINT`: 8 byte (64-bit)
- UUID: 16 byte (128-bit)

Artinya, UUID membutuhkan dua kali lipat ruang penyimpanan dibanding `BIGINT`.

Namun, jika menggunakan tipe kolom `UUID` bawaan di PostgreSQL, penyimpanan ini sudah dioptimalkan menjadi 16 byte (bukan string 36 karakter seperti `"550e8400-e29b-41d4-a716-446655440000"`). Jadi tetap efisien, meskipun lebih besar dari integer.

Dampaknya tidak hanya ke storage, tapi juga ke:

- ukuran index yang lebih besar,
- cache yang lebih cepat penuh,
- dan potensi penurunan performa query.

#### 2. Random insertion problem

Masalah terbesar UUID acak bukan di ukuran, tapi di cara data dimasukkan ke dalam index.

Berbeda dengan integer auto-increment yang selalu naik secara berurutan, UUID acak tidak memiliki urutan yang jelas. Nilainya bisa berada di mana saja dalam rentang yang sangat besar.

Akibatnya, saat data baru dimasukkan:

- database harus mencari posisi yang tepat di dalam struktur B-tree,
- sering kali harus menyisipkan data di tengah-tengah node,
- dan ini memicu proses split (pemecahan node) serta rebalancing (penyusunan ulang tree).

Ilustrasinya kira-kira seperti ini:

```
B-tree dengan UUID acak:
        [m3d...]
       /        \
  [a1b...]    [z9f...]
     ↑ insert [c7e...] di sini → harus rebalance!
```

Berbeda dengan integer yang selalu “nambah di ujung”, UUID acak membuat struktur B-tree lebih sering “diacak ulang”.

Dampaknya:

- performa insert menjadi lebih lambat,
- fragmentasi index meningkat,
- dan dalam skala besar, bisa mempengaruhi performa keseluruhan sistem.

#### Perbedaan dampak di PostgreSQL vs MySQL

Masalah ini sebenarnya terjadi di semua database yang menggunakan B-tree, tapi tingkat dampaknya berbeda.

- Di PostgreSQL, efeknya masih relatif terkendali karena tabel dan index dipisah secara fisik.
- Di MySQL (terutama dengan InnoDB), primary key digunakan sebagai clustered index, artinya data tabel disimpan mengikuti urutan primary key.

Akibatnya, pada MySQL:

- UUID acak bisa menyebabkan data di disk tersebar tidak berurutan,
- lebih banyak page split,
- dan performa bisa turun lebih signifikan dibanding PostgreSQL.

#### Ringkasan pemahaman

UUID menawarkan keunggulan dalam hal keunikan global dan fleksibilitas, terutama untuk sistem terdistribusi. Namun, ada trade-off yang harus diperhatikan:

- ukuran lebih besar dibanding integer,
- dan performa insert yang lebih rendah akibat sifatnya yang acak.

Karena itu, penggunaan UUID sebagai primary key sebaiknya dipertimbangkan dengan konteks kebutuhan. Jika tidak benar-benar membutuhkan keunikan global atau distributed ID generation, integer (khususnya `BIGINT`) biasanya tetap menjadi pilihan yang lebih efisien.

### c. UUID v7 dan ULID — Solusi Terbaik Jika Harus Pakai UUID

Kalau kamu memang perlu menggunakan UUID, ada pendekatan yang jauh lebih “ramah performa” dibanding UUID acak (seperti v4), yaitu menggunakan UUID v7 atau ULID.

Keduanya dirancang untuk mengatasi masalah utama UUID sebelumnya—terutama soal urutan data di dalam index.

#### Apa itu UUID v7?

UUID v7 adalah versi UUID yang bersifat **time-ordered**. Artinya, sebagian bit di awal nilainya menyimpan informasi waktu (timestamp).

Implikasinya sederhana tapi penting:

- UUID yang dibuat lebih baru akan memiliki nilai yang lebih besar.
- Data akan masuk ke index secara berurutan, bukan acak.

Dengan kata lain, UUID v7 mencoba meniru perilaku integer auto-increment, tetapi tetap mempertahankan keunggulan UUID: bisa digenerate tanpa bergantung pada database.

#### Apa itu ULID?

ULID (Universally Unique Lexicographically Sortable Identifier) punya konsep yang sangat mirip.

Strukturnya terdiri dari dua bagian:

- timestamp di bagian awal,
- diikuti oleh nilai acak (randomness) untuk menjaga keunikan.

Contohnya:

```
01ARZ3NDEKTSV4RRFFQ69G5FAV
|---------||--------------|
 timestamp     randomness
 (48 bit)      (80 bit)
```

Karena timestamp berada di depan, ULID bisa diurutkan secara leksikografis (berdasarkan urutan string), dan hasilnya tetap sesuai dengan urutan waktu.

#### Kenapa UUID v7 dan ULID lebih baik?

Masalah terbesar UUID sebelumnya adalah **random insertion** di B-tree. UUID v7 dan ULID menghilangkan masalah ini dengan membuat ID yang **bertambah seiring waktu**.

Dampaknya:

- Insert data menjadi lebih terprediksi (mirip seperti integer).
- Minim operasi split dan rebalancing pada B-tree.
- Performa insert jauh lebih stabil dibanding UUID acak.

Jadi, kamu mendapatkan kombinasi terbaik:

- keunikan global (seperti UUID),
- tanpa penalti performa besar,
- dan tetap bisa di-generate di luar database.

#### Kapan UUID atau ULID benar-benar dibutuhkan?

Dalam banyak aplikasi, integer (`BIGINT`) tetap pilihan terbaik. Tapi ada skenario tertentu di mana UUID/ULID menjadi sangat berguna, terutama dalam pola **optimistic UI**.

Optimistic UI adalah pendekatan di mana aplikasi langsung menampilkan hasil ke user, tanpa menunggu respon dari server.

Alurnya kira-kira seperti ini:

```
Alur Optimistic UI:
1. User klik "Buat Entri Baru"
2. Client generate UUID/ULID secara lokal
3. UI langsung menampilkan entri dengan ID tersebut
4. Data dikirim ke server/database di background
5. Database menyimpan dengan ID yang sama
```

Di sini, client sudah punya ID sejak awal, sehingga:

- tidak perlu menunggu database,
- UI terasa lebih cepat dan responsif,
- dan konflik ID hampir tidak mungkin terjadi.

Sebaliknya, jika menggunakan `BIGINT` auto-increment:

- ID baru hanya tersedia setelah data berhasil disimpan di database,
- sehingga client harus menunggu respon server sebelum bisa menampilkan data dengan ID yang valid.

#### Ringkasan pemahaman

UUID v7 dan ULID adalah evolusi dari UUID klasik yang memperbaiki kelemahan utamanya.

Mereka cocok digunakan ketika:

- kamu butuh ID yang bisa dibuat di sisi client,
- sistem bersifat terdistribusi,
- atau ingin menghindari bottleneck pada database.

Namun, jika kebutuhan tersebut tidak ada, `BIGINT` auto-increment tetap menjadi solusi paling sederhana, efisien, dan optimal secara performa.

### d. Masalah Keamanan Integer Primary Key

Menggunakan integer berurutan sebagai primary key memang sangat efisien dari sisi performa dan struktur database. Tapi ketika ID ini terekspos ke publik—misalnya lewat URL atau API—muncul potensi masalah dari sisi keamanan.

#### Kenapa integer bisa jadi masalah?

Karena sifatnya yang berurutan dan mudah ditebak.

Contoh sederhana:

```
GET /invoices/1042
```

Dari satu angka ini saja, seseorang bisa menebak beberapa hal:

- Kemungkinan besar sudah ada sekitar 1.042 invoice di sistem.
- Mereka bisa mencoba mengakses ID lain seperti `1041`, `1043`, dan seterusnya.

Teknik ini dikenal sebagai _incrementing attack_—menebak resource lain hanya dengan menaikkan atau menurunkan angka ID.

Dampaknya bisa serius:

- Data sensitif bisa terekspos jika tidak ada validasi akses yang kuat.
- Informasi internal seperti jumlah user, jumlah transaksi, atau volume data bisa “bocor” hanya dari pola ID.

Perlu dicatat: ini bukan berarti integer primary key itu salah, tapi cara kita mengeksposnya ke publik yang perlu diperhatikan.

#### Solusi yang direkomendasikan: Secondary Public ID

Pendekatan terbaik bukan mengganti primary key, melainkan **menambahkan layer ID kedua yang khusus untuk publik**.

Artinya:

- Tetap gunakan `BIGINT` sebagai primary key internal (untuk performa dan efisiensi).
- Tambahkan kolom baru sebagai **public ID** yang bersifat acak dan sulit ditebak.

Salah satu cara populer adalah menggunakan Nano ID.

Nano ID menghasilkan string pendek, acak, dan sangat sulit diprediksi, misalnya:

```
V1StGXR8_Z5jdHi6B-myT
```

Dengan pendekatan ini:

- ID internal tetap optimal untuk database.
- ID publik aman digunakan di URL atau API.

#### Contoh implementasi

```sql id="m9r1k2"
CREATE TABLE invoices (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, -- internal
    public_id  TEXT UNIQUE NOT NULL DEFAULT nanoid(),           -- publik
    amount     NUMERIC,
    -- ...
);
```

Penjelasan:

- `id`: digunakan oleh database sebagai primary key utama.
- `public_id`: digunakan oleh client (frontend, API consumer, dll).
- `DEFAULT nanoid()`: setiap baris otomatis mendapatkan ID publik yang unik.

#### Bagaimana alurnya di aplikasi?

Di sisi user atau API, yang digunakan adalah `public_id`, bukan `id`.

Contohnya:

```
URL publik: GET /invoices/V1StGXR8_Z5jdHi6B-myT
                                ↓
Database query: SELECT * FROM invoices
                WHERE public_id = 'V1StGXR8_Z5jdHi6B-myT'
```

Jadi:

- User tidak pernah melihat ID internal.
- Tidak ada pola yang bisa ditebak.
- Serangan berbasis enumerasi menjadi jauh lebih sulit.

#### Kenapa ini lebih baik daripada mengganti ke UUID?

Mengganti primary key langsung ke UUID memang bisa menyelesaikan masalah “tebakan ID”, tapi membawa trade-off lain:

- ukuran index lebih besar,
- performa insert bisa menurun (terutama jika random UUID),
- dan kompleksitas tambahan.

Dengan pendekatan **secondary public ID**, kamu mendapatkan:

- keamanan dari sisi publik,
- tanpa mengorbankan performa internal database.

#### Ringkasan pemahaman

Integer primary key tetap pilihan terbaik untuk performa, tapi tidak ideal jika langsung diekspos ke publik.

Solusi yang lebih seimbang adalah:

- gunakan `BIGINT` sebagai primary key internal,
- tambahkan `public_id` yang acak untuk kebutuhan eksternal.

Dengan begitu, kamu menjaga performa database tetap optimal sekaligus meningkatkan keamanan dari sisi akses publik.

## 3. Hubungan Antar Konsep

Keempat konsep di atas membentuk alur pengambilan keputusan (_decision tree_) yang kohesif:

```
Apakah kamu butuh generate ID tanpa koordinasi database?
(misal: optimistic UI, distributed system)
        │
       YES → Gunakan UUID v7 atau ULID (time-ordered)
        │
        NO
        │
        ▼
Gunakan BIGINT GENERATED ALWAYS AS IDENTITY (auto-increment)
        │
        ▼
Apakah ID ini akan tampil di URL/API publik?
        │
       YES → Tambahkan kolom public_id (Nano ID) di samping BIGINT
        │
        NO
        │
        ▼
Selesai! BIGINT sudah cukup.
```

Inti dari seluruh diskusi adalah **trade-off antara tiga hal**:

- **Performa insert** → BIGINT unggul karena sequential, UUID acak merusak B-tree.
- **Kemudahan generate ID tanpa DB** → UUID/ULID unggul.
- **Keamanan / privasi** → UUID memang lebih obscure, tapi Nano ID sebagai secondary key adalah solusi yang lebih bersih.

---

## 4. Kesimpulan

| Skenario                                      | Rekomendasi                                             |
| --------------------------------------------- | ------------------------------------------------------- |
| Kasus umum (98%)                              | `BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY`       |
| Butuh ID sebelum simpan ke DB (optimistic UI) | UUID v7 atau ULID                                       |
| UUID random (v4)                              | ❌ Hindari sebagai primary key (random insertion buruk) |
| Khawatir ID integer bocor di URL/API          | Tambahkan kolom `public_id` dengan Nano ID              |

**Tiga rekomendasi utama pembicara:**

1. **Gunakan `BIGINT`** sebagai primary key untuk mayoritas kasus.
2. **Jika harus pakai UUID**, gunakan varian **UUID v7** atau **ULID** yang time-ordered, bukan UUID acak.
3. **Jika khawatir keamanan**, jangan ganti primary key — cukup **tambahkan Nano ID** sebagai public-facing identifier di samping BIGINT.
