# 📝 Composite Indexes (Index Multi-Kolom) di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang **composite index** (index gabungan), yaitu index yang mencakup lebih dari satu kolom sekaligus. Materi mencakup cara membuat composite index, aturan penggunaannya — terutama konsep **leftmost prefix** — serta bagaimana PostgreSQL secara internal menelusuri B-tree untuk memanfaatkan index tersebut secara optimal. Pembicara juga menunjukkan perbedaan performa antara direct B-tree traversal, index scan, dan table scan melalui contoh nyata.

---

## 2. Konsep Utama

### a. Apa Itu Composite Index?

#### Konsep Dasar

**Composite index** adalah index yang dibuat menggunakan **lebih dari satu kolom sekaligus** dalam satu struktur index. Jadi, berbeda dengan index biasa (single index) yang hanya fokus pada satu kolom, composite index menggabungkan beberapa kolom menjadi satu kesatuan.

Tujuannya adalah untuk meningkatkan performa query yang melibatkan **lebih dari satu kondisi** atau yang membutuhkan proses **sorting berdasarkan beberapa kolom**.

#### Kenapa Perlu Composite Index?

Bayangkan kamu sering menjalankan query seperti ini:

```sql
SELECT * FROM users
WHERE first_name = 'John'
AND last_name = 'Doe'
AND birthday = '1990-01-01';
```

Kalau kamu hanya punya index terpisah di masing-masing kolom (`first_name`, `last_name`, `birthday`), database harus menggabungkan hasil dari beberapa index tersebut. Ini bisa jadi kurang efisien, apalagi jika datanya besar.

Di sinilah composite index lebih unggul: database bisa langsung menggunakan satu index yang sudah mencakup semua kolom tersebut.

#### Perbandingan Single Index vs Composite Index

Contoh berikut memperlihatkan perbedaannya:

```sql
-- Index tunggal (kurang optimal jika ada banyak kolom yang diquery bersamaan)
CREATE INDEX idx_firstname ON users (first_name);

-- Composite index (lebih optimal)
CREATE INDEX multi ON users USING BTREE (first_name, last_name, birthday);
```

Penjelasannya:

- **Index tunggal (`idx_firstname`)** hanya membantu saat query menggunakan `first_name` saja.
- **Composite index (`multi`)** bisa digunakan untuk query yang melibatkan:
  - `first_name`
  - `first_name + last_name`
  - `first_name + last_name + birthday`

Artinya, semakin lengkap urutan kolom yang digunakan sesuai dengan definisi index, semakin optimal performanya.

#### Catatan Penting (Urutan Kolom)

Composite index sangat bergantung pada **urutan kolom**. Dalam contoh:

```sql
(first_name, last_name, birthday)
```

Maka index akan efektif untuk:

- `WHERE first_name = ...`
- `WHERE first_name = ... AND last_name = ...`
- `WHERE first_name = ... AND last_name = ... AND birthday = ...`

Namun, **tidak optimal** untuk:

- `WHERE last_name = ...` saja
- `WHERE birthday = ...` saja

Karena database membaca index dari kolom pertama terlebih dahulu (dikenal sebagai prinsip _leftmost prefix_).

#### Kapan Harus Menggunakan Composite Index?

Gunakan composite index ketika:

- Query sering melibatkan **beberapa kolom sekaligus**
- Ada kebutuhan **sorting berdasarkan lebih dari satu kolom**
- Pola query relatif konsisten (kolom yang dipakai tidak berubah-ubah)

Dengan penggunaan yang tepat, composite index bisa memberikan peningkatan performa yang signifikan dibandingkan kombinasi index tunggal.

### b. Kemampuan Bawaan PostgreSQL: Bitmap Index Scan

#### Apa Itu Bitmap Index Scan?

Salah satu keunggulan dari PostgreSQL adalah kemampuannya untuk tetap mengoptimalkan query, bahkan ketika **tidak tersedia composite index yang sesuai**.

Dalam situasi seperti ini, PostgreSQL bisa menggunakan teknik yang disebut **_bitmap index scan_**. Teknik ini memungkinkan database untuk **menggabungkan hasil dari beberapa index tunggal** menjadi satu hasil yang efisien.

#### Cara Kerjanya Secara Sederhana

Misalnya kamu punya dua index terpisah:

```sql id="x1k2a"
CREATE INDEX idx_firstname ON users (first_name);
CREATE INDEX idx_lastname ON users (last_name);
```

Lalu kamu menjalankan query seperti ini:

```sql id="p9z3lm"
SELECT * FROM users
WHERE first_name = 'John'
AND last_name = 'Doe';
```

Karena tidak ada composite index `(first_name, last_name)`, PostgreSQL tidak langsung menyerah. Sebagai gantinya, dia akan:

1. Menggunakan index `idx_firstname` untuk mencari semua baris dengan `first_name = 'John'`
2. Menggunakan index `idx_lastname` untuk mencari semua baris dengan `last_name = 'Doe'`
3. Menggabungkan kedua hasil tersebut dalam bentuk **bitmap** (semacam representasi biner dari baris yang cocok)
4. Mengambil irisan (intersection) dari kedua bitmap tersebut untuk mendapatkan hasil akhir

Dengan cara ini, PostgreSQL tetap bisa menghindari full table scan dan menjaga performa tetap baik.

#### Kenapa Ini Menarik?

Tidak semua database memiliki fitur seperti ini. Banyak sistem lain hanya bisa memilih satu index saja, atau bahkan langsung fallback ke full scan jika tidak ada index yang cocok.

Di sini, PostgreSQL punya keunggulan karena bisa:

- Menggunakan **lebih dari satu index dalam satu query**
- Menggabungkan hasilnya secara efisien
- Tetap menjaga performa meskipun struktur index belum ideal

#### Tapi Tetap, Composite Index Lebih Unggul

Walaupun bitmap index scan sangat membantu, ini bukan berarti kamu bisa mengabaikan composite index.

Composite index yang dirancang dengan baik tetap lebih cepat karena:

- Tidak perlu menggabungkan beberapa hasil index
- Struktur datanya sudah sesuai dengan kebutuhan query
- Lebih efisien untuk operasi **sorting (ORDER BY)**

Sebagai gambaran, bitmap index scan adalah solusi “pintar” saat kondisi belum optimal, sedangkan composite index adalah solusi “ideal” yang memang sudah disiapkan dari awal.

#### Kapan Bitmap Index Scan Digunakan?

Biasanya PostgreSQL akan memilih bitmap index scan ketika:

- Tidak ada composite index yang cocok
- Query melibatkan beberapa kondisi (`AND` / `OR`)
- Menggunakan beberapa index tunggal masih lebih efisien dibanding full table scan

Dengan memahami ini, kamu bisa lebih bijak dalam mendesain index: tahu kapan harus mengandalkan kemampuan bawaan PostgreSQL, dan kapan perlu membuat composite index untuk performa maksimal.

### c. Aturan Leftmost Prefix

#### Konsep Inti

Dalam penggunaan composite index, ada satu prinsip yang sangat penting untuk dipahami, yaitu **aturan leftmost prefix**. Aturan ini menentukan apakah sebuah query bisa memanfaatkan index secara optimal atau tidak.

Ada dua aturan dasar yang perlu kamu ingat:

1. **Left to right, no skipping**
   Artinya, kolom dalam index harus digunakan **berurutan dari kiri ke kanan**, tanpa melewati kolom di tengah.

2. **Stops at the first range**
   Artinya, penggunaan index akan **berhenti** ketika menemui kondisi range seperti `>`, `<`, atau `BETWEEN`. Setelah titik itu, kolom berikutnya dalam index tidak lagi dimanfaatkan secara optimal.

> Penting: urutan yang dimaksud di sini adalah **urutan kolom saat index dibuat**, bukan urutan penulisan kondisi di query.

#### Contoh Struktur Index

Misalnya kita punya composite index seperti ini:

```sql id="k2n8sd"
CREATE INDEX multi ON users USING BTREE (first_name, last_name, birthday);
```

Urutan kolomnya adalah:

1. `first_name`
2. `last_name`
3. `birthday`

Ini berarti database akan membaca index mulai dari `first_name`, lalu ke `last_name`, dan terakhir `birthday`.

#### Contoh Penggunaan dan Analisis

Mari kita lihat beberapa contoh query dan bagaimana index digunakan:

```sql id="a1x9qp"
-- ❌ Tidak menggunakan index (skip kolom pertama)
EXPLAIN SELECT * FROM users WHERE last_name = 'Francis';
```

Penjelasan:
Query ini **tidak bisa menggunakan composite index** karena langsung loncat ke `last_name` tanpa menyertakan `first_name`. Ini melanggar aturan _left to right, no skipping_.

```sql id="b7m4zt"
-- ✅ Menggunakan index (mulai dari kolom paling kiri)
EXPLAIN SELECT * FROM users WHERE first_name = 'Aaron';
```

Penjelasan:
Query ini bisa menggunakan index karena dimulai dari kolom paling kiri (`first_name`). Walaupun hanya satu kolom, index tetap valid digunakan.

```sql id="c3r8vn"
-- ✅ Menggunakan index secara lebih optimal
EXPLAIN SELECT * FROM users
WHERE first_name = 'Aaron' AND last_name = 'Francis';
```

Penjelasan:
Ini adalah penggunaan yang lebih optimal karena:

- Mengikuti urutan kolom dari kiri ke kanan
- Tidak ada kolom yang dilewati

Database bisa memanfaatkan index sampai kolom `last_name`.

#### Contoh Kasus “Stops at the First Range”

Sekarang bayangkan query seperti ini:

```sql id="d6p2ky"
EXPLAIN SELECT * FROM users
WHERE first_name = 'Aaron'
AND last_name > 'F'
AND birthday = '1990-01-01';
```

Penjelasan:

- `first_name = 'Aaron'` → masih optimal
- `last_name > 'F'` → ini adalah kondisi **range**
- Setelah kondisi range ini, penggunaan index **berhenti**

Artinya:

- Kolom `birthday` **tidak akan digunakan dalam index**, meskipun ada di definisi index

#### Insight Penting

Dari sini kamu bisa melihat bahwa:

- Urutan kolom dalam composite index itu sangat menentukan
- Tidak semua query otomatis “cocok” dengan index yang ada
- Kesalahan kecil seperti melewati kolom pertama bisa membuat index tidak terpakai sama sekali

Karena itu, saat mendesain composite index, kamu perlu menyesuaikannya dengan pola query yang paling sering digunakan. Dengan begitu, aturan leftmost prefix ini bisa dimanfaatkan secara maksimal.

### d. Urutan dalam Query vs. Urutan dalam Index

#### Konsep Utama

Salah satu hal yang sering membingungkan adalah perbedaan antara:

- **urutan kondisi dalam query**, dan
- **urutan kolom dalam index**

Banyak yang mengira bahwa urutan penulisan kondisi di `WHERE` harus sama persis dengan urutan index. Padahal, itu tidak benar.

Dalam PostgreSQL, urutan kondisi di query **tidak berpengaruh terhadap penggunaan index**. PostgreSQL memiliki query planner yang cukup pintar untuk mengatur ulang kondisi agar tetap optimal.

Yang benar-benar menentukan adalah **urutan kolom saat index dibuat**.

#### Contoh Kasus

Misalnya kita punya composite index seperti ini:

```sql id="q1x2zl"
CREATE INDEX multi ON users USING BTREE (first_name, last_name);
```

Sekarang perhatikan dua query berikut:

```sql id="m4n8yk"
-- Query 1
SELECT * FROM users
WHERE first_name = 'Aaron' AND last_name = 'Francis';

-- Query 2
SELECT * FROM users
WHERE last_name = 'Francis' AND first_name = 'Aaron';
```

#### Apakah Hasilnya Berbeda?

Tidak. Kedua query ini dianggap **setara** oleh PostgreSQL.

Penjelasannya:

- Walaupun urutan kondisi berbeda, PostgreSQL akan mengoptimalkan query tersebut di belakang layar
- Selama kedua kolom (`first_name` dan `last_name`) digunakan, dan **urutan index dimulai dari `first_name`**, maka index tetap bisa dimanfaatkan secara optimal

#### Insight Penting

Yang perlu kamu pegang di sini:

- Urutan di `WHERE` → **fleksibel**, tidak perlu terlalu dipikirkan
- Urutan di `INDEX` → **krusial**, harus dirancang sesuai pola query

Dengan kata lain, kamu bebas menulis query dengan gaya apa pun, tetapi saat membuat index, kamu harus berpikir strategis: kolom mana yang paling sering muncul dulu dalam kondisi query.

#### Hubungan dengan Leftmost Prefix

Konsep ini sebenarnya masih berkaitan erat dengan aturan _leftmost prefix_ yang sudah dibahas sebelumnya.

Walaupun urutan di query bisa ditukar-tukar, PostgreSQL tetap akan:

- Mengidentifikasi kolom mana yang sesuai dengan urutan index
- Memastikan penggunaan index tetap mengikuti urutan dari kiri ke kanan

Jadi, fleksibilitas di query tidak mengubah fakta bahwa **struktur index tetap menjadi acuan utama dalam optimasi performa**.

### e. Perilaku PostgreSQL Saat Leftmost Prefix Tidak Terbentuk Sempurna

#### Gambaran Umum

Secara teori, kalau aturan _leftmost prefix_ tidak terpenuhi, index seharusnya tidak bisa digunakan. Tapi dalam praktiknya, PostgreSQL punya pendekatan yang lebih fleksibel.

PostgreSQL tetap berusaha **memanfaatkan sebagian index**, meskipun ada kolom di tengah yang “dilewati”. Hanya saja, cara kerjanya berubah—tidak lagi seefisien traversal langsung pada struktur B-tree.

#### Ilustrasi Kasus

Misalkan kita punya composite index:

```sql id="i9k3lm"
CREATE INDEX multi ON users USING BTREE (first_name, last_name, birthday);
```

Perhatikan query berikut:

```sql id="z7x2qp"
EXPLAIN SELECT * FROM users
WHERE first_name = 'Aaron'
AND birthday = '1989-02-14';
```

Di sini, kolom `last_name` yang berada di tengah **tidak digunakan**. Ini berarti aturan _leftmost prefix_ tidak terpenuhi secara sempurna.

#### Apa yang Dilakukan PostgreSQL?

Alih-alih mengabaikan index sepenuhnya, PostgreSQL tetap mencoba memanfaatkannya dengan langkah berikut:

1. **Menelusuri B-tree berdasarkan kolom pertama (`first_name`)**
   PostgreSQL akan dengan cepat menemukan semua baris yang memiliki `first_name = 'Aaron'`. Bagian ini masih efisien.

2. **Masuk ke leaf nodes hasil pencarian tersebut**
   Setelah semua data dengan `first_name = 'Aaron'` ditemukan, PostgreSQL berada di bagian leaf dari index.

3. **Melakukan scanning tambahan untuk filter berikutnya**
   Karena `last_name` dilewati, PostgreSQL **tidak bisa langsung menggunakan `birthday` dalam struktur index**.
   Akibatnya, database harus:
   - Men-scan semua entri dengan `first_name = 'Aaron'`
   - Lalu menyaring mana yang `birthday = '1989-02-14'`

Proses ini disebut sebagai **index scan parsial**, bukan traversal B-tree yang optimal.

#### Dampaknya ke Performa

Pendekatan ini jelas masih lebih baik dibanding full table scan, tapi:

- Lebih lambat dibanding penggunaan composite index yang “pas”
- Semakin banyak data dengan `first_name = 'Aaron'`, semakin besar biaya scanning tambahan

#### Solusi yang Lebih Optimal

Sekarang bandingkan jika kita membuat index yang lebih sesuai dengan pola query:

```sql id="u4n8vc"
CREATE INDEX multi2 ON users USING BTREE (first_name, birthday);
```

Dengan index ini:

- Urutan kolom langsung cocok dengan query
- Tidak ada kolom yang “menghalangi” di tengah
- PostgreSQL bisa melakukan traversal B-tree secara penuh tanpa scanning tambahan

Akibatnya, untuk query sebelumnya, PostgreSQL akan **memilih `multi2` dibanding `multi`** karena jauh lebih efisien.

#### Insight Penting

Dari kasus ini, ada beberapa pelajaran yang bisa kamu ambil:

- PostgreSQL memang “pintar”, tapi tetap ada batasannya
- Melewati kolom di tengah tidak membuat index sepenuhnya gagal, tapi performanya turun
- Desain composite index harus benar-benar mengikuti pola query yang sering digunakan

Dengan memahami perilaku ini, kamu bisa lebih strategis dalam menentukan urutan kolom di index, bukan sekadar mengandalkan optimasi otomatis dari database.

### f. Tiga Tingkat Efisiensi Query

#### Gambaran Besar

Saat database menjalankan sebuah query, ada beberapa cara yang bisa dipilih untuk mengambil data. Di PostgreSQL, pilihan ini sangat dipengaruhi oleh apakah index bisa digunakan secara optimal atau tidak.

Secara umum, ada **tiga tingkat efisiensi** dalam eksekusi query—dari yang paling cepat sampai yang paling lambat.

#### 🥇 Direct B-tree Traversal (Paling Optimal)

Ini adalah kondisi terbaik.

Terjadi ketika:

- Composite index digunakan secara **penuh**
- Aturan _leftmost prefix_ terpenuhi dengan sempurna
- Tidak ada kolom yang dilewati
- Tidak terhenti oleh kondisi range di tengah

Contoh:

```sql id="a9f2kd"
CREATE INDEX multi ON users USING BTREE (first_name, last_name, birthday);

EXPLAIN SELECT * FROM users
WHERE first_name = 'Aaron'
AND last_name = 'Francis';
```

Penjelasan:

- Query mengikuti urutan index dari kiri ke kanan
- PostgreSQL bisa langsung “menelusuri” struktur B-tree tanpa hambatan
- Tidak perlu scanning tambahan

Hasilnya: **sangat cepat dan efisien**

#### 🥈 Index Scan (Cukup Efisien)

Ini adalah kondisi menengah—masih menggunakan index, tapi tidak sepenuhnya optimal.

Terjadi ketika:

- Hanya sebagian index yang bisa digunakan
- Ada kolom yang dilewati
- Atau ada kondisi yang membuat traversal berhenti lebih awal

Contoh:

```sql id="k3l8vn"
EXPLAIN SELECT * FROM users
WHERE first_name = 'Aaron'
AND birthday = '1989-02-14';
```

Penjelasan:

- PostgreSQL menggunakan `first_name` untuk masuk ke index
- Tapi karena `last_name` dilewati, index tidak bisa langsung digunakan untuk `birthday`
- Akhirnya database harus melakukan **scan di leaf nodes** untuk menyaring hasil

Hasilnya:

- Lebih cepat dari full table scan
- Tapi lebih lambat dibanding direct traversal

#### 🥉 Table Scan / Sequential Scan (Paling Lambat)

Ini adalah kondisi terburuk.

Terjadi ketika:

- Tidak ada index yang cocok
- Atau query tidak bisa memanfaatkan index sama sekali

Contoh:

```sql id="z1p4xs"
EXPLAIN SELECT * FROM users
WHERE last_name = 'Francis';
```

(dengan asumsi tidak ada index pada `last_name`)

Penjelasan:

- PostgreSQL tidak punya “jalan pintas”
- Harus membaca seluruh isi tabel dari awal sampai akhir

Hasilnya:

- Sangat lambat, terutama jika jumlah data besar

#### Insight Penting

Dari tiga tingkat ini, kamu bisa melihat pola yang jelas:

- Semakin sesuai query dengan struktur index → semakin cepat
- Semakin banyak “kompromi” (skip kolom, range, dll) → performa turun
- Tanpa index → performa paling buruk

Karena itu, tujuan utama saat mendesain index adalah **mendorong query agar masuk ke kategori pertama (direct B-tree traversal)** sebisa mungkin.

### g. Strategi Menentukan Urutan Kolom dalam Composite Index

#### Kenapa Urutan Kolom Itu Penting?

Dalam composite index, urutan kolom bukan sekadar detail teknis—ini adalah faktor utama yang menentukan apakah index bisa digunakan secara optimal atau tidak.

Di PostgreSQL, index akan dibaca dari **kiri ke kanan**. Artinya, kolom yang berada di posisi paling kiri punya pengaruh paling besar terhadap performa query.

Kalau urutannya salah, index bisa jadi:

- Tidak terpakai sama sekali, atau
- Hanya terpakai sebagian (kurang optimal)

#### Prinsip Dasar Penentuan Urutan

Ada dua aturan praktis yang bisa kamu pegang:

1. **Kolom yang paling sering digunakan dalam query → letakkan di kiri**
2. **Kolom yang jarang digunakan atau bersifat range → letakkan di kanan**

Kenapa begitu?

- Kolom di kiri menentukan apakah query bisa “masuk” ke index dengan cepat
- Kolom di kanan biasanya hanya membantu penyaringan lanjutan atau sorting
- Kondisi range (`>`, `<`, `BETWEEN`) bisa menghentikan penggunaan index, jadi lebih aman diletakkan di belakang

#### Contoh Kasus

Misalnya kamu punya situasi seperti ini:

- Banyak query menggunakan `first_name`
- Hanya sedikit query yang menggunakan `birthday`

Maka index yang direkomendasikan adalah:

```sql id="n5x2qd"
CREATE INDEX idx ON users (first_name, birthday);
```

Bukan seperti ini:

```sql id="w8k1mz"
CREATE INDEX idx ON users (birthday, first_name);
```

#### Analisis Kenapa Ini Penting

Dengan urutan `(first_name, birthday)`:

- Query seperti:

  ```sql
  WHERE first_name = 'Aaron'
  ```

  → bisa langsung menggunakan index

- Query seperti:

  ```sql
  WHERE first_name = 'Aaron' AND birthday = '1990-01-01'
  ```

  → bisa menggunakan index secara lebih lengkap

Sebaliknya, jika urutannya `(birthday, first_name)`:

- Query yang hanya menggunakan `first_name` **tidak bisa menggunakan index**
- Bahkan query kombinasi pun bisa jadi kurang optimal jika tidak dimulai dari `birthday`

#### Insight Tambahan

Saat menentukan urutan kolom, kamu sebenarnya sedang menjawab pertanyaan ini:

> “Kolom mana yang paling sering jadi pintu masuk query?”

Kolom itulah yang seharusnya berada di posisi paling kiri.

Dengan strategi ini, kamu tidak hanya membuat index “benar secara teknis”, tapi juga **relevan dengan pola query nyata** di aplikasi. Ini yang membedakan antara index yang sekadar ada, dan index yang benar-benar meningkatkan performa.

## 3. Hubungan Antar Konsep

Pemahaman tentang composite index dibangun secara bertahap:

1. **Single-column index** adalah fondasi — sudah baik, tetapi terbatas jika query melibatkan banyak kolom.
2. **Composite index** memperluas konsep tersebut dengan menggabungkan beberapa kolom dalam satu struktur B-tree yang terurut.
3. **Leftmost prefix rule** adalah kunci memahami _kapan_ dan _seberapa_ index dimanfaatkan — urutan deklarasi index menentukan segalanya.
4. **Perilaku internal PostgreSQL** (B-tree traversal vs. index scan) menjelaskan _mengapa_ aturan leftmost prefix begitu penting: B-tree hanya bisa ditelusuri secara efisien jika dimulai dari kolom paling kiri.
5. **Strategi desain index** adalah penerapan praktis dari semua aturan di atas — memilih kolom mana yang harus di kiri dan mana di kanan berdasarkan pola query yang paling umum.
6. Aturan **"stops at the first range"** adalah kelanjutan dari aturan leftmost prefix yang akan dibahas di video berikutnya, melengkapi gambaran utuh tentang bagaimana composite index bekerja.

---

## 4. Kesimpulan

- **Composite index** lebih powerful daripada beberapa single-column index terpisah, terutama untuk query multi-kondisi dan sorting.
- Aturan utama: **left to right, no skipping, stops at the first range** — urutan deklarasi index sangat krusial.
- **Urutan kondisi dalam query tidak penting**, yang penting adalah urutan kolom saat index dibuat.
- PostgreSQL punya optimasi bawaan untuk membantu bahkan ketika leftmost prefix tidak terbentuk sempurna, tetapi hasilnya (index scan) tetap lebih lambat dari direct B-tree traversal.
- **Desain index yang baik**: letakkan kolom yang paling sering difilter di posisi paling kiri, dan kolom yang lebih jarang atau bersifat range di posisi paling kanan.
