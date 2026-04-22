# 📝 Composite Index & Range Condition di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas bagaimana PostgreSQL menggunakan **composite index** (index dengan lebih dari satu kolom) ketika query mengandung **range condition** (kondisi kisaran seperti `<`, `>`, `BETWEEN`). Konsep kunci yang dijelaskan adalah aturan **"kiri ke kanan, tanpa lompat, berhenti di range pertama"** — sebuah prinsip penting dalam merancang index yang efisien di database berbasis B-tree.

---

## 2. Konsep Utama

### a. Composite Index (Index Multi-Kolom)

Composite index adalah index yang dibuat dari **lebih dari satu kolom** dalam satu struktur index. Berbeda dengan index biasa yang hanya fokus pada satu kolom, composite index menyusun beberapa kolom **secara berurutan** dalam satu jalur pencarian.

Konsep penting yang harus dipahami di sini adalah: **urutan kolom itu bukan sekadar formalitas — tapi menentukan bagaimana database membaca dan memanfaatkan index tersebut.**

#### Kenapa urutan kolom itu penting?

Bayangkan composite index seperti daftar yang diurutkan berdasarkan beberapa kriteria:

1. Urutkan berdasarkan kolom pertama
2. Jika nilainya sama, lanjut ke kolom kedua
3. Jika masih sama, lanjut ke kolom ketiga

Artinya, database seperti PostgreSQL akan **menggunakan index dari kiri ke kanan** (left-to-right rule). Ini sering disebut sebagai _leftmost prefix rule_.

Jadi, kalau urutan kolom tidak sesuai dengan pola query, index bisa jadi tidak optimal — bahkan bisa saja tidak digunakan sama sekali.

#### Contoh pembuatan composite index

Misalnya kita punya dua index seperti berikut:

```sql
-- Index 1: first_name -> last_name -> birthday
CREATE INDEX first_last_birth ON users (first_name, last_name, birthday);

-- Index 2: first_name -> birthday -> last_name
CREATE INDEX first_birth_last ON users (first_name, birthday, last_name);
```

Sekilas terlihat sama karena memakai kolom yang sama, tapi sebenarnya **perilakunya berbeda jauh** saat digunakan dalam query.

#### Cara membaca perbedaan kedua index

**Index 1 (`first_name, last_name, birthday`)**

Index ini cocok digunakan untuk query seperti:

```sql
SELECT * FROM users
WHERE first_name = 'John'
  AND last_name = 'Doe';
```

Atau:

```sql
SELECT * FROM users
WHERE first_name = 'John'
  AND last_name = 'Doe'
  AND birthday = '1990-01-01';
```

Kenapa bisa optimal? Karena query mengikuti urutan kolom dalam index dari kiri ke kanan.

Namun, kalau query seperti ini:

```sql
SELECT * FROM users
WHERE last_name = 'Doe';
```

Index **tidak bisa digunakan secara optimal**, karena kita melewati kolom pertama (`first_name`).

#### Index 2 (`first_name, birthday, last_name`)

Sekarang lihat index kedua. Ini akan lebih cocok untuk query seperti:

```sql
SELECT * FROM users
WHERE first_name = 'John'
  AND birthday = '1990-01-01';
```

Tetapi untuk query yang membutuhkan `last_name` sebelum `birthday`, performanya bisa menurun karena urutannya tidak sesuai.

#### Bagaimana PostgreSQL memilih index?

PostgreSQL cukup pintar untuk memilih index yang paling efisien berdasarkan query yang dijalankan. Ia akan melihat:

- Kolom apa saja yang digunakan di `WHERE`
- Urutan kolom dalam index
- Statistik data (jumlah data, distribusi nilai, dll)

Namun, ini bukan berarti kita bisa asal membuat index. Kalau desain index-nya tidak sesuai pola query yang sering digunakan, PostgreSQL tetap tidak bisa “menyelamatkan” performanya.

#### Intinya

- Composite index memungkinkan pencarian lebih cepat untuk query yang melibatkan beberapa kolom.
- Urutan kolom dalam index **sangat menentukan** apakah index bisa digunakan atau tidak.
- Selalu sesuaikan urutan kolom dalam index dengan pola query yang paling sering digunakan.

Kalau diibaratkan, composite index itu seperti buku yang punya beberapa kategori pengurutan — kalau kamu mencari sesuai urutan yang benar, hasilnya cepat. Tapi kalau urutannya tidak cocok, kamu tetap harus “mencari manual” lagi.

### b. Cara Kerja B-Tree Index pada Equality Condition

Untuk memahami bagian ini, kita perlu melihat bagaimana **B-Tree index** bekerja saat menghadapi kondisi query, khususnya kondisi **equality (`=`)**.

Secara sederhana, B-Tree adalah struktur data berbentuk pohon yang terurut. Karena datanya sudah tersusun rapi, PostgreSQL bisa melakukan pencarian dengan cara **menelusuri cabang pohon secara langsung (direct traversal)**, bukan dengan memeriksa satu per satu data seperti pada full scan.

#### Apa yang terjadi saat menggunakan equality (`=`)?

Ketika query menggunakan kondisi seperti `column = value`, PostgreSQL bisa:

- Masuk ke root node B-Tree
- Membandingkan nilai
- Langsung memilih cabang kiri atau kanan
- Terus turun sampai menemukan node yang tepat

Proses ini sangat cepat karena setiap langkah langsung mengarah ke lokasi yang lebih spesifik. Tidak ada “tebakan”, semuanya berbasis struktur yang sudah terurut.

#### Contoh query

```sql
SELECT * FROM users
WHERE first_name = 'Aaron'
  AND last_name = 'Francis'
  AND birthday < '1989-12-31';
```

Mari kita bedah cara PostgreSQL mengeksekusi query ini, khususnya jika ada composite index seperti `(first_name, last_name, birthday)`.

#### Tahap 1: Equality pada `first_name`

Untuk kondisi:

```sql
first_name = 'Aaron'
```

PostgreSQL akan langsung melakukan traversal di B-Tree untuk menemukan posisi nilai `'Aaron'`. Karena ini equality, prosesnya sangat efisien — langsung turun ke node yang relevan tanpa scanning.

#### Tahap 2: Equality pada `last_name`

Setelah menemukan semua entri dengan `first_name = 'Aaron'`, PostgreSQL lanjut ke:

```sql
last_name = 'Francis'
```

Karena ini masih equality dan masih sesuai urutan index, database bisa **melanjutkan traversal di dalam cabang yang sudah sempit tadi**. Artinya, pencarian semakin spesifik dan tetap efisien.

Sampai di sini, performanya masih optimal karena kita masih berada dalam pola **left-to-right + equality**.

#### Tahap 3: Range pada `birthday`

Sekarang masuk ke kondisi:

```sql
birthday < '1989-12-31'
```

Di sinilah perilakunya berubah.

Berbeda dengan equality, kondisi **range (`<`, `>`, `BETWEEN`, dll)** tidak bisa lagi menggunakan direct traversal untuk mempersempit satu titik. Kenapa? Karena hasilnya bukan satu nilai, tapi **sekumpulan nilai dalam rentang tertentu**.

Akibatnya:

- PostgreSQL berhenti melakukan traversal “loncat langsung”
- Database mulai melakukan **scan dalam range** dari titik yang sudah ditemukan sebelumnya

Dengan kata lain, setelah menemukan `first_name = 'Aaron'` dan `last_name = 'Francis'`, PostgreSQL akan:

- Menemukan titik awal di B-Tree
- Lalu membaca data secara berurutan selama masih memenuhi kondisi `birthday < '1989-12-31'`

#### Insight penting

- Equality (`=`) memungkinkan **direct traversal** → sangat cepat
- Range (`<`, `>`, dll) mengubah strategi jadi **range scan**
- Begitu masuk ke kondisi range, kolom setelahnya dalam index biasanya tidak bisa dimanfaatkan secara optimal

#### Intinya

- B-Tree index bekerja paling efisien saat menggunakan kondisi equality berurutan dari kiri ke kanan
- Selama masih menggunakan `=`, PostgreSQL bisa terus “menyusuri pohon” dengan presisi tinggi
- Begitu masuk kondisi range, traversal berhenti dan berubah menjadi scanning dalam rentang

Cara berpikirnya sederhana: equality itu seperti mencari alamat rumah yang spesifik, sedangkan range itu seperti diminta mencari semua rumah di satu jalan — tetap terarah, tapi tidak bisa langsung ke satu titik saja.

### c. Apa yang Terjadi Saat Menjumpai Range Condition

Sampai titik ini, kita sudah melihat bahwa kondisi **equality (`=`)** memungkinkan PostgreSQL melakukan pencarian dengan sangat presisi melalui B-Tree. Tapi situasinya berubah ketika query mulai melibatkan **range condition** seperti `<`, `>`, atau `BETWEEN`.

Di sinilah penting untuk memahami perubahan strategi yang dilakukan database.

#### Perubahan dari traversal ke scan

Begitu PostgreSQL menemukan **range condition pertama** dalam urutan index, ia **tidak bisa lagi melakukan direct traversal** seperti sebelumnya.

Kenapa? Karena range tidak menunjuk ke satu nilai spesifik, melainkan ke **sekumpulan nilai**. Akibatnya:

- PostgreSQL berhenti “meloncat” antar node secara langsung
- Dan mulai membaca data secara berurutan dalam rentang tertentu

Strategi ini disebut sebagai **range scan**.

#### Apa itu mode scan?

Mode scan berarti PostgreSQL akan:

- Menentukan **titik awal** berdasarkan hasil traversal sebelumnya
- Lalu membaca entri index **secara berurutan (sequential dalam index)**
- Berhenti ketika batas kondisi range sudah tidak terpenuhi

Jadi meskipun tetap menggunakan index, cara kerjanya sudah tidak seefisien traversal berbasis equality.

#### Ilustrasi alur kerja

Misalnya kita punya index seperti:

```
[B-Tree Index: first_name | last_name | birthday]
```

Dan query dengan kombinasi equality + range, maka alurnya seperti ini:

1. PostgreSQL melakukan traversal untuk mencari `first_name = 'Aaron'`
2. Dilanjutkan traversal untuk `last_name = 'Francis'`
3. Sampai di titik node `"Aaron Francis"` → ini menjadi **starting point**
4. Dari titik ini, PostgreSQL mulai **scan ke arah kanan** pada index
5. Semua entri dengan `birthday < '1989-12-31'` akan dibaca satu per satu
6. Untuk setiap hasil yang cocok, PostgreSQL akan mengambil data aslinya dari **heap** (penyimpanan utama tabel)

#### Peran heap dalam proses ini

Perlu dipahami bahwa index hanya menyimpan:

- Nilai kolom yang diindex
- Dan pointer ke lokasi data di tabel utama (heap)

Jadi setelah menemukan kandidat dari index, PostgreSQL tetap perlu:

- Mengakses heap
- Mengambil baris lengkap yang sesuai

Proses ini sering disebut sebagai **heap lookup**.

#### Insight penting

- Range condition adalah titik perubahan dari traversal ke scan
- Semakin lebar range, semakin banyak data yang harus discan
- Setelah range digunakan, kolom berikutnya dalam index biasanya tidak bisa dimanfaatkan secara optimal

#### Intinya

- Equality membantu PostgreSQL menemukan titik dengan cepat
- Range menentukan seberapa banyak data yang harus dibaca setelah titik tersebut ditemukan
- Kombinasi keduanya tetap efisien, tapi performanya sangat bergantung pada seberapa sempit range yang digunakan

Cara paling mudah membayangkannya:
Traversal itu seperti langsung menuju alamat rumah tertentu, sedangkan range scan itu seperti berjalan menyusuri satu jalan dan memeriksa rumah satu per satu sampai batas tertentu.

### d. Aturan Emas: "Left to Right, No Skipping, Stops at the First Range"

Dalam penggunaan composite index berbasis B-Tree, ada satu prinsip penting yang hampir selalu jadi acuan dalam memahami performa query. Prinsip ini sering diringkas sebagai:

**“Left to right, no skipping, stops at the first range.”**

Kalimat ini bukan sekadar teori — ini adalah cara PostgreSQL benar-benar bekerja saat memanfaatkan index multi-kolom.

#### Left to Right

PostgreSQL selalu membaca dan menggunakan kolom dalam index **dari kiri ke kanan**, sesuai urutan saat index dibuat.

Artinya, kalau index didefinisikan seperti ini:

```sql id="lhw7e3"
(first_name, last_name, birthday)
```

Maka urutan aksesnya adalah:

1. `first_name`
2. `last_name`
3. `birthday`

Database tidak bisa langsung “loncat” ke kolom kedua atau ketiga tanpa melalui kolom sebelumnya. Semua proses selalu dimulai dari kolom paling kiri.

#### No Skipping

Masih terkait dengan urutan, PostgreSQL **tidak bisa melewati kolom** dalam index.

Kalau ada kolom dalam index yang tidak digunakan dalam `WHERE clause`, maka traversal akan **berhenti di situ**.

Contoh:

```sql id="6n6mxs"
WHERE last_name = 'Francis'
```

Dengan index `(first_name, last_name, birthday)`, query ini tidak bisa langsung memanfaatkan index secara optimal, karena kolom pertama (`first_name`) dilewati.

Akibatnya, PostgreSQL kemungkinan akan beralih ke strategi lain seperti scan, karena tidak bisa melakukan traversal yang terarah dari awal.

#### Stops at the First Range

Ini bagian yang paling sering jadi jebakan.

Begitu PostgreSQL menemukan **range condition pertama** (misalnya `<`, `>`, `BETWEEN`), maka:

- Proses traversal berhenti
- Berubah menjadi **range scan**
- Kolom setelahnya dalam index **tidak bisa lagi digunakan untuk mempersempit pencarian secara langsung**

Jadi meskipun ada kolom tambahan di index, manfaatnya tidak maksimal setelah range terjadi.

#### Contoh praktis

Misalnya kita punya index:

```sql id="tdp6um"
(first_name, last_name, birthday)
```

Query berikut termasuk efisien:

```sql id="4g4aq8"
WHERE first_name = 'Aaron'          -- traverse langsung
  AND last_name = 'Francis'         -- traverse langsung
  AND birthday < '1989-12-31';      -- mulai scan (range pertama)
```

Alurnya jelas:

- `first_name` → traversal
- `last_name` → traversal
- `birthday` → mulai range scan

Karena range berada di **kolom terakhir**, semua kolom sebelumnya masih bisa dimanfaatkan secara optimal.

Sekarang bandingkan dengan kasus berikut:

```sql id="gksu8q"
-- Index: (first_name, birthday, last_name)

WHERE first_name = 'Aaron'          -- traverse langsung
  AND birthday < '1989-12-31'       -- range pertama → langsung scan
  AND last_name = 'Francis';        -- tidak bisa dimanfaatkan optimal
```

Di sini masalahnya ada pada posisi range.

Begitu PostgreSQL menemui `birthday < ...`, proses traversal langsung berhenti. Akibatnya:

- `last_name` tidak bisa lagi digunakan untuk mempersempit hasil secara efisien
- Database harus memfilter hasil scan lebih banyak

#### Intinya

- Urutan kolom dalam composite index sangat menentukan performa
- PostgreSQL hanya bisa bekerja optimal jika query mengikuti urutan tersebut
- Jangan melewati kolom (no skipping), karena traversal akan terhenti
- Letakkan kondisi range di bagian akhir agar tidak “memutus” efisiensi index terlalu cepat

Kalau dipikirkan, aturan ini seperti jalur navigasi: kamu harus mulai dari titik awal, tidak bisa lompat-lompat, dan begitu masuk area “range”, kamu harus menjelajah lebih luas tanpa bantuan navigasi yang presisi lagi.

### e. Mengapa PostgreSQL Memilih `first_last_birth` bukan `first_birth_last`?

Untuk memahami keputusan PostgreSQL dalam memilih index, kita perlu menghubungkannya dengan prinsip yang sudah dibahas sebelumnya: **urutan kolom, equality, dan range condition**.

Mari kita lihat kembali contoh query berikut:

```sql id="k2m1zp"
WHERE first_name = 'Aaron'
  AND last_name = 'Francis'
  AND birthday < '1989-12-31'
```

Sekarang kita bandingkan dua index yang tersedia:

```sql id="y6g9rf"
-- Index A
(first_name, last_name, birthday)

-- Index B
(first_name, birthday, last_name)
```

#### Analisis Index `first_last_birth`

Pada index ini, urutannya adalah:

```sql id="8o6t3l"
(first_name → last_name → birthday)
```

Ketika query dijalankan:

1. `first_name = 'Aaron'` → bisa langsung di-traverse
2. `last_name = 'Francis'` → masih bisa di-traverse (lanjutan dari kolom sebelumnya)
3. `birthday < '1989-12-31'` → mulai range scan

Artinya, PostgreSQL bisa memanfaatkan **dua kondisi equality terlebih dahulu** sebelum masuk ke range.

Efeknya:

- Pencarian menjadi sangat sempit sejak awal
- Jumlah data yang perlu di-scan pada tahap range menjadi lebih sedikit
- Secara keseluruhan → **lebih efisien**

#### Analisis Index `first_birth_last`

Sekarang lihat urutan index kedua:

```sql id="m1c4vz"
(first_name → birthday → last_name)
```

Eksekusi query akan berjalan seperti ini:

1. `first_name = 'Aaron'` → traversal masih optimal
2. `birthday < '1989-12-31'` → langsung masuk range scan
3. `last_name = 'Francis'` → **tidak bisa digunakan untuk traversal lagi**

Masalah utamanya ada di langkah kedua.

Begitu PostgreSQL menemukan range pada `birthday`, proses traversal berhenti. Akibatnya:

- Kolom `last_name` tidak bisa dimanfaatkan untuk mempersempit pencarian di level index
- Database harus memfilter `last_name` setelah proses scan berjalan
- Jumlah data yang diproses jadi lebih banyak

Hasil akhirnya → **kurang efisien dibanding index pertama**

#### Kenapa PostgreSQL memilih yang lebih efisien?

PostgreSQL memiliki query planner yang secara otomatis:

- Menganalisis struktur query
- Membandingkan index yang tersedia
- Mengestimasi biaya (cost) dari masing-masing opsi

Dari situ, PostgreSQL akan memilih index yang:

- Mengurangi jumlah data yang harus dibaca
- Memanfaatkan traversal semaksimal mungkin sebelum masuk ke range

Dalam kasus ini, `first_last_birth` jelas lebih unggul karena:

- Dua kolom pertama menggunakan equality (paling efisien)
- Range diletakkan di akhir (tidak mengganggu traversal sebelumnya)

#### Intinya

- Posisi range dalam urutan index sangat menentukan efisiensi
- Semakin banyak kondisi equality yang bisa digunakan sebelum range, semakin baik
- PostgreSQL akan otomatis memilih index yang memberikan jalur pencarian paling sempit dan hemat biaya

Dengan kata lain, desain index yang baik bukan hanya soal kolom apa yang dipakai, tapi **bagaimana urutannya mendukung pola query yang dijalankan**.

### f. Perbedaan Perilaku PostgreSQL vs Database Lain

Saat membahas composite index, penting untuk memahami bahwa **tidak semua database bekerja dengan cara yang sama**. Cara PostgreSQL memanfaatkan index—terutama saat ada _range condition_ atau kolom yang dilewati—cukup berbeda dibandingkan beberapa database lain, khususnya versi lama.

#### Perilaku di banyak database lain

Pada beberapa database (misalnya MySQL versi lama), ada aturan yang lebih “kaku” dalam penggunaan index. Biasanya:

- Begitu query menemui **range condition pertama** (`<`, `>`, `BETWEEN`),
- Atau ada kolom dalam index yang **tidak digunakan (skip)**,

Maka **penggunaan index akan berhenti di titik tersebut**.

Artinya:

- Kolom setelah range atau skip **tidak akan digunakan sama sekali**
- Bahkan dalam beberapa kasus, database bisa memilih untuk **tidak menggunakan index sama sekali** dan beralih ke full scan

Pendekatan ini membuat performa sangat bergantung pada urutan kolom yang tepat. Sedikit saja tidak sesuai, manfaat index bisa hilang drastis.

#### Perilaku PostgreSQL

PostgreSQL punya pendekatan yang lebih fleksibel.

Meskipun tetap mengikuti prinsip:

- left-to-right
- no skipping
- stops at the first range

PostgreSQL **tidak langsung “menyerah”** ketika menemui range atau kolom yang dilewati.

Sebaliknya, PostgreSQL akan:

- Tetap menggunakan index yang ada
- Tapi beralih dari mode **traversal** ke **scan**

Artinya:

- Index masih dipakai sebagai titik awal pencarian
- Namun proses selanjutnya dilakukan dengan membaca data dalam rentang tertentu (range scan)
- Kolom setelahnya mungkin tetap membantu dalam filtering, walaupun tidak seefisien traversal

#### Dampak praktisnya

Pendekatan ini membuat PostgreSQL:

- Lebih **toleran terhadap desain index yang kurang ideal**
- Tetap bisa memberikan performa yang cukup baik dalam banyak kasus
- Tidak langsung jatuh ke full table scan hanya karena urutan kolom tidak sempurna

Namun, ini bukan berarti kita bisa mengabaikan desain index.

#### Intinya

- Database lain cenderung **berhenti total** dalam memanfaatkan index setelah range atau skip
- PostgreSQL tetap menggunakan index, tapi dalam bentuk **scan, bukan traversal**
- Ini membuat PostgreSQL lebih fleksibel, tapi **efisiensi terbaik tetap hanya tercapai dengan desain index yang tepat**

Cara mudah membayangkannya:
Database lain seperti GPS yang langsung mati kalau jalannya tidak sesuai rute, sedangkan PostgreSQL tetap memberi arah — hanya saja kamu harus berjalan lebih manual setelah keluar dari jalur optimal.

## 3. Hubungan Antar Konsep

```
Composite Index
      │
      ▼
Urutan Kolom Menentukan Efisiensi
      │
      ├──► Equality Condition → Direct Traversal (sangat cepat)
      │
      └──► Range Condition → Switch ke Scan (lebih lambat)
                  │
                  └──► Kolom setelah range: tidak dapat di-traverse,
                       hanya di-filter dari hasil scan

Aturan Desain Index yang Optimal:
  [Equality columns] → [Less-used equality] → [Range column]
  (paling kiri)                              (paling kanan)
```

Hubungan antar konsep dapat dirangkum sebagai berikut:

1. **Composite index** memberi PostgreSQL kemampuan untuk menyaring data menggunakan beberapa kolom sekaligus.
2. **B-tree traversal** (direct traversal) hanya bisa dilakukan selama kondisinya adalah **equality** (`=`), dibaca dari kiri ke kanan tanpa melewati kolom.
3. Saat **range condition** ditemui, traversal berhenti dan beralih ke **scan**, yang tetap menggunakan index tetapi kurang efisien.
4. Oleh karena itu, **desain index yang optimal** menempatkan kolom dengan equality condition yang paling sering digunakan di **sisi kiri**, dan range condition di **sisi kanan**.

---

## 4. Kesimpulan

Prinsip utama yang harus selalu diingat saat merancang composite index di PostgreSQL adalah:

> **"Left to right, no skipping, stops at the first range."**

- Tempatkan kolom dengan **equality condition** yang paling sering digunakan di **paling kiri**.
- Tempatkan **range condition** di **paling kanan** dari index.
- Menghindari melewati kolom (skipping) dan menempatkan range di tengah akan menjaga traversal tetap efisien.
- PostgreSQL lebih unggul dari banyak database lain karena tetap menggunakan index bahkan setelah menjumpai skip atau range, meski dalam mode scan yang kurang efisien.

Memahami aturan ini adalah kunci untuk menulis query yang cepat dan merancang schema database yang performatif.
