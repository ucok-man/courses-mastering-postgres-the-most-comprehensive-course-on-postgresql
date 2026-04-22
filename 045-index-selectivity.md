# 📝 Index Selectivity: Cardinality & Selectivity dalam Database Indexing

## 1. Ringkasan Singkat

Video ini membahas bagaimana menentukan apakah sebuah kolom merupakan kandidat yang baik untuk dibuatkan _index_. Tidak semua kolom layak di-_index_ — kuncinya ada pada dua konsep penting: **cardinality** (jumlah nilai unik dalam kolom) dan **selectivity** (rasio nilai unik terhadap total baris). Video juga menunjukkan cara menghitung kedua metrik ini secara langsung dengan SQL, serta menjelaskan kapan Postgres memilih untuk menggunakan atau mengabaikan sebuah _index_.

---

## 2. Konsep Utama

### a. Mengapa Tidak Semua Kolom Cocok untuk Di-_index_

Sebelum memutuskan membuat _index_, ada satu pertanyaan penting yang harus selalu kita tanyakan:

> _“Apakah kolom ini benar-benar membantu mempersempit hasil pencarian?”_

Ini bukan sekadar pertanyaan formal — ini inti dari efektivitas sebuah _index_.

#### Memahami Tujuan Utama Index

Secara sederhana, _index_ dibuat untuk mempercepat proses pencarian data. Caranya adalah dengan **menghindari scan seluruh tabel** (_full table scan_) dan langsung mengarah ke baris yang relevan.

Namun, ini hanya efektif jika kolom yang di-_index_ memiliki **variasi nilai yang cukup** (sering disebut _selectivity_).

#### Contoh Kasus: Index yang Tidak Berguna

Bayangkan kita memiliki tabel `users`, dan semua data di dalamnya memiliki nilai yang sama untuk kolom `first_name`, misalnya:

```sql
first_name = 'Aaron'
```

Lalu kita membuat _index_ pada kolom tersebut dan menjalankan query:

```sql
SELECT * FROM users WHERE first_name = 'Aaron';
```

Sekilas terlihat benar — kita menggunakan _index_ untuk mencari data. Tapi jika dipikirkan lebih dalam:

- Semua baris memiliki nilai `'Aaron'`
- Artinya, query akan tetap mengembalikan **seluruh isi tabel**
- _Database engine_ tidak mendapatkan keuntungan apa pun dari _index_ tersebut

Dalam kondisi seperti ini, _index_ tidak membantu sama sekali. Bahkan, dalam beberapa kasus, _database_ bisa saja **mengabaikan index** dan tetap melakukan _full table scan_ karena dianggap lebih efisien.

#### Intinya: Selectivity Itu Penting

Kunci dari efektivitas _index_ adalah **kemampuan kolom untuk mempersempit hasil pencarian**.

Bandingkan dua kondisi berikut:

- Kolom dengan nilai unik atau jarang berulang (misalnya: email, ID) → **cocok untuk index**
- Kolom dengan nilai yang hampir selalu sama (misalnya: status = 'active' untuk hampir semua user) → **tidak efektif untuk index**

Semakin sedikit baris yang cocok dengan kondisi query, semakin besar manfaat _index_.

#### Cara Berpikir yang Tepat

Saat ingin membuat _index_, biasakan berpikir seperti ini:

- Apakah kolom ini sering digunakan dalam `WHERE`, `JOIN`, atau `ORDER BY`?
- Apakah nilai di kolom ini cukup bervariasi?
- Apakah query akan mengambil sebagian kecil data, bukan hampir seluruh tabel?

Jika jawabannya “ya”, maka kolom tersebut kandidat yang baik untuk di-_index_. Jika tidak, sebaiknya pertimbangkan kembali — karena _index_ juga punya biaya (penyimpanan dan overhead saat insert/update).

Dengan pendekatan ini, kita tidak hanya membuat _index_, tapi membuat _index_ yang benar-benar berguna.

### b. Cardinality

Saat membahas apakah sebuah kolom layak di-_index_ atau tidak, salah satu konsep penting yang sering muncul adalah **cardinality**.

#### Apa Itu Cardinality?

**Cardinality** adalah jumlah nilai unik (_distinct_) yang terdapat dalam suatu kolom.

Semakin banyak variasi nilai dalam kolom tersebut, semakin tinggi cardinality-nya.

#### Contoh Sederhana

Mari kita lihat beberapa ilustrasi:

- Kolom Boolean (`true` / `false`)
  → hanya memiliki 2 kemungkinan nilai
  → **cardinality = 2**

- Kolom `birthday` pada tabel `users`
  → berisi banyak variasi tanggal lahir
  → bisa memiliki ribuan nilai unik
  → misalnya **~11.000 nilai unik dari 1 juta baris**

#### Menghitung Cardinality dengan Query

Kita bisa menghitung cardinality secara langsung menggunakan SQL, misalnya:

```sql
-- Menghitung cardinality kolom birthday
SELECT COUNT(*)
FROM (SELECT DISTINCT birthday FROM users) AS subquery;
-- Hasil: ~10.900 nilai unik
```

Penjelasannya:

- `SELECT DISTINCT birthday` akan mengambil semua nilai tanggal lahir yang unik
- Hasilnya dijadikan subquery
- Lalu `COUNT(*)` menghitung jumlah baris unik tersebut

Hasil akhirnya menunjukkan berapa banyak variasi nilai dalam kolom `birthday`.

#### Kenapa Cardinality Penting?

Cardinality memberi kita gambaran awal tentang **seberapa “beragam” data dalam kolom**.

- Cardinality tinggi → banyak nilai unik → potensi _index_ lebih efektif
- Cardinality rendah → sedikit variasi → potensi _index_ kurang efektif

Namun, ini baru sebatas indikasi awal.

#### Jangan Terjebak: Cardinality Bukan Satu-Satunya Faktor

Penting untuk dipahami bahwa:

> **Cardinality saja belum cukup untuk menentukan apakah sebuah index itu bagus atau tidak.**

Beberapa hal yang perlu diingat:

- Kolom dengan cardinality tinggi **belum tentu** selalu cocok untuk _index_
  (misalnya jika jarang digunakan dalam query)
- Kolom dengan cardinality rendah **tidak selalu buruk**
  (tergantung bagaimana kolom tersebut digunakan dalam filter atau kombinasi index)

#### Cara Berpikir yang Lebih Tepat

Anggap cardinality sebagai **indikator awal**, bukan keputusan akhir.

Saat mengevaluasi kolom untuk di-_index_, kita tetap perlu mempertimbangkan:

- Pola query (apakah sering dipakai di `WHERE`, `JOIN`, dll)
- Distribusi data (apakah nilai tersebar merata atau tidak)
- Kombinasi dengan kolom lain (misalnya dalam composite index)

Dengan begitu, kita tidak hanya melihat angka cardinality, tapi memahami konteks penggunaannya secara menyeluruh.

### c. Selectivity

Setelah memahami **cardinality**, langkah berikutnya adalah memahami konsep yang lebih “tajam” untuk menilai kualitas kolom dalam _index_, yaitu **selectivity**.

#### Apa Itu Selectivity?

**Selectivity** adalah **rasio** antara jumlah nilai unik (_distinct_) dengan total jumlah baris dalam tabel.

Secara sederhana, rumusnya:

```
Selectivity = Jumlah Nilai Distinct / Total Jumlah Baris
```

Nilai selectivity selalu berada di antara **0 sampai 1**, dan di sinilah kita mulai bisa menilai seberapa efektif sebuah kolom jika dijadikan _index_.

#### Cara Membaca Nilai Selectivity

- Nilai mendekati **1**
  → artinya hampir setiap baris punya nilai unik
  → **sangat selektif**
  → _index_ biasanya **sangat efektif**

- Nilai mendekati **0**
  → artinya banyak baris memiliki nilai yang sama
  → **tidak selektif**
  → _index_ cenderung **kurang berguna**

Dengan kata lain, selectivity membantu menjawab pertanyaan:

> _“Seberapa besar kemungkinan sebuah kondisi query benar-benar menyaring data?”_

#### Contoh Perhitungan dengan SQL

Mari kita lihat bagaimana menghitung selectivity langsung dari database.

```sql id="1h3o9d"
-- Menghitung selectivity kolom birthday
SELECT
  CAST(COUNT(DISTINCT birthday) AS DECIMAL) / COUNT(*) AS selectivity
FROM users;
-- Hasil: ~0.0111 (kurang dari 2% baris bisa dibedakan)
```

Penjelasan:

- `COUNT(DISTINCT birthday)` → jumlah nilai unik
- `COUNT(*)` → total jumlah baris
- Hasil pembagian → nilai selectivity

Nilai `~0.0111` berarti hanya sekitar **1.11% variasi unik dibanding total data** — artinya banyak baris berbagi nilai yang sama.

Sekarang bandingkan dengan kolom `id`:

```sql id="s3g8qx"
-- Selectivity kolom id (primary key) = SEMPURNA
SELECT
  CAST(COUNT(DISTINCT id) AS DECIMAL) / COUNT(*) AS selectivity
FROM users;
-- Hasil: 1.0000 (setiap baris memiliki nilai unik)
```

Di sini hasilnya **1.0**, yang berarti:

- Setiap baris memiliki nilai yang berbeda
- Ini adalah kondisi **ideal** untuk sebuah _index_

#### Perbandingan Nyata

Berikut gambaran sederhana untuk membandingkan beberapa kolom:

- `id` → **1.0000**
  Setiap baris unik → _index_ sangat efektif

- `birthday` → **~0.0111**
  Ada variasi, tapi masih banyak duplikasi → cukup efektif dalam beberapa kasus

- `is_pro` → **sangat kecil (~0.000...2)**
  Hanya punya 2 nilai (`true/false`) → hampir tidak membantu menyaring data

#### Insight Penting

Di sinilah selectivity lebih “berguna” dibanding cardinality saja.

- Cardinality hanya memberi tahu **berapa banyak nilai unik**
- Selectivity memberi tahu **seberapa signifikan nilai unik itu dibanding total data**

Dua kolom bisa saja punya cardinality sama, tapi jika ukuran tabel berbeda, selectivity-nya bisa sangat berbeda.

#### Cara Berpikir yang Tepat

Saat mengevaluasi kolom untuk di-_index_, gunakan selectivity sebagai panduan praktis:

- Jika query biasanya mengembalikan **sebagian kecil data** → selectivity tinggi → cocok untuk _index_
- Jika query sering mengembalikan **sebagian besar data** → selectivity rendah → _index_ kurang membantu

Dengan memahami selectivity, kita tidak hanya melihat “berapa banyak nilai unik”, tetapi juga **seberapa efektif kolom tersebut dalam benar-benar menyaring data saat query dijalankan**.

### d. Kasus Khusus: Data yang _Skewed_ (Tidak Merata)

Sejauh ini kita belajar bahwa **selectivity tinggi = index efektif**, dan sebaliknya. Tapi di dunia nyata, ada satu kasus penting yang sering “mengecoh” jika kita hanya melihat angka global, yaitu ketika data **tidak terdistribusi secara merata** (_skewed_).

#### Apa Itu Data Skewed?

Data disebut **skewed** ketika distribusinya tidak seimbang — ada nilai yang sangat sering muncul, dan ada juga yang sangat jarang.

Dalam kondisi seperti ini, melihat selectivity **secara keseluruhan kolom** saja bisa menyesatkan.

#### Contoh Kasus: Kolom Boolean `is_pro`

Secara teori:

- Kolom Boolean hanya punya 2 nilai (`true` / `false`)
- Cardinality sangat rendah
- Selectivity keseluruhan juga rendah
  → Sekilas terlihat **tidak cocok untuk index**

Namun, mari kita lihat distribusi datanya:

```sql id="jrt8a6"
SELECT COUNT(*) FROM users WHERE is_pro = true;
-- Hasil: ~44.000 baris (dari ~1.000.000)
-- Artinya hanya ~4.4% pengguna yang adalah pro member
```

Artinya:

- `true` hanya muncul di sebagian kecil data
- Sisanya (~95.6%) adalah `false`

Ini adalah contoh klasik data yang **skewed**.

#### Menghitung Selectivity untuk Nilai Tertentu

Sekarang kita fokus hanya pada nilai `true`:

```sql id="l9z0o3"
-- Selectivity khusus untuk nilai 'true'
SELECT
  CAST(COUNT(*) AS DECIMAL(17,14)) / (SELECT COUNT(*) FROM users)
FROM users
WHERE is_pro = true;
-- Hasil: jauh lebih baik dari selectivity keseluruhan kolom
```

Hasil ini menunjukkan bahwa:

- Hanya sebagian kecil baris yang memenuhi kondisi `is_pro = true`
- Query dengan kondisi tersebut akan mengambil data dalam jumlah terbatas

#### Kenapa Index Jadi Berguna?

Di sinilah insight pentingnya:

Jika kita sering menjalankan query seperti:

```sql id="f3yq2k"
SELECT * FROM users WHERE is_pro = true;
```

Maka:

- Tanpa index → database harus scan ~1.000.000 baris
- Dengan index → database bisa langsung menuju ~44.000 baris yang relevan

Ini adalah **penyempitan data yang signifikan**, meskipun secara keseluruhan kolom tersebut terlihat “buruk” dari sisi selectivity.

#### Cara Berpikir yang Lebih Akurat

Kasus ini mengajarkan bahwa:

- Jangan hanya melihat **selectivity global**
- Perhatikan juga **distribusi nilai di dalam kolom**

Gunakan pendekatan seperti ini:

- Apakah ada nilai tertentu yang jarang muncul?
- Apakah query sering memfilter nilai tersebut?
- Apakah hasil query hanya sebagian kecil dari total data?

Jika jawabannya “ya”, maka _index_ tetap bisa sangat efektif — bahkan pada kolom dengan cardinality rendah.

#### Kesimpulan Penting

> Perhitungan selectivity secara keseluruhan hanya akurat jika distribusi data **normal (merata)**.
> Jika data **skewed**, kita perlu melihat lebih dalam — khususnya pada nilai yang sering digunakan dalam query.

Dengan memahami ini, kita bisa menghindari keputusan yang terlalu “mekanis” dan mulai berpikir seperti database optimizer: melihat data bukan hanya dari angka, tapi juga dari **pola distribusinya**.

### e. Selectivity pada Kondisi Range (Query dengan `>` atau `<`)

Selama ini kita banyak membahas selectivity dalam konteks pencarian nilai yang spesifik (misalnya `=`). Tapi dalam praktik, query sering menggunakan **kondisi range** seperti `>` atau `<`. Di sini, konsep selectivity tetap berlaku — hanya saja cara berpikirnya sedikit berbeda.

#### Apa yang Berbeda pada Query Range?

Pada query seperti:

- `birthday > '1989-02-14'`
- `birthday < '1989-02-14'`

Kita tidak lagi mencari satu nilai tertentu, melainkan **sekumpulan nilai dalam rentang tertentu**. Artinya, jumlah baris yang dikembalikan bisa sangat bervariasi tergantung posisi nilai tersebut dalam distribusi data.

#### Contoh Kasus Nyata

Perhatikan dua query berikut:

```sql id="gk7p2x"
-- Query 1: birthday SETELAH 1989-02-14
EXPLAIN SELECT * FROM users WHERE birthday > '1989-02-14';
-- Postgres TIDAK menggunakan index
-- Karena query ini mengembalikan ~572.000 baris (> 50% tabel)
```

```sql id="z9v3qd"
-- Query 2: birthday SEBELUM 1989-02-14
EXPLAIN SELECT * FROM users WHERE birthday < '1989-02-14';
-- Postgres MENGGUNAKAN index
-- Karena query ini mengembalikan ~417.000 baris (< 50% tabel)
```

Sekilas terlihat aneh, karena kedua query menggunakan kolom yang sama dan operator yang mirip. Tapi keputusan database berbeda. Kenapa?

#### Cara Database Berpikir

Database seperti PostgreSQL tidak sekadar melihat “ada index atau tidak”, tapi juga menghitung **berapa banyak baris yang akan diambil**.

- Pada query pertama (`>`):
  - Mengembalikan ~572.000 baris
  - Itu lebih dari setengah tabel
  - Menggunakan index justru dianggap tidak efisien
    → **lebih cepat langsung scan seluruh tabel**

- Pada query kedua (`<`):
  - Mengembalikan ~417.000 baris
  - Kurang dari setengah tabel
  - Index masih memberikan keuntungan
    → **index digunakan**

#### Intuisi di Balik Keputusan Ini

Menggunakan index bukan selalu lebih cepat.

Index membantu ketika:

- Kita hanya mengambil **sebagian kecil data**
- Sehingga database bisa “melompat” langsung ke bagian yang relevan

Namun jika:

- Data yang diambil terlalu banyak (misalnya >50%)
- Maka biaya menggunakan index + akses ke tabel bisa lebih mahal dibanding membaca tabel secara langsung

#### Tentang “Batas 50%”

Sering disebut bahwa:

> Jika hasil query lebih dari setengah tabel, maka index tidak digunakan.

Tapi ini **bukan aturan mutlak**.

PostgreSQL menggunakan:

- Statistik distribusi data
- Estimasi biaya (_cost-based optimizer_)
- Faktor seperti I/O, caching, dan ukuran tabel

Angka “50%” hanyalah **rule of thumb** untuk membantu kita memahami pola umumnya.

#### Cara Berpikir yang Tepat

Untuk query range, biasakan berpikir seperti ini:

- Seberapa besar rentang data yang diambil?
- Apakah hasilnya sebagian kecil atau sebagian besar dari tabel?
- Bagaimana distribusi nilai di kolom tersebut?

Karena dua query yang terlihat mirip bisa menghasilkan performa yang sangat berbeda — tergantung **seberapa selektif kondisi range-nya**.

Dengan memahami ini, kita bisa lebih bijak dalam membuat index dan tidak menganggap bahwa semua kondisi `>` atau `<` otomatis akan mendapat manfaat dari index.

### f. Statistik Internal Postgres

Saat PostgreSQL memutuskan apakah akan menggunakan _index_ atau tidak, ia **tidak benar-benar menjalankan query-nya terlebih dahulu** untuk mengukur performa. Sebaliknya, PostgreSQL menggunakan pendekatan yang lebih efisien: **mengandalkan statistik internal** tentang data yang ada di tabel.

#### Apa Itu Statistik Internal?

Statistik ini berisi informasi penting seperti:

- Distribusi nilai dalam kolom
- Perkiraan jumlah nilai unik (_cardinality_)
- Pola penyebaran data (apakah merata atau _skewed_)

Dengan informasi ini, PostgreSQL bisa **mengestimasi biaya (cost)** dari berbagai strategi eksekusi query, lalu memilih yang paling efisien — misalnya antara menggunakan _index_ atau melakukan _full table scan_.

#### Dari Mana Statistik Ini Didapat?

Statistik tidak muncul begitu saja. PostgreSQL mengumpulkannya melalui dua mekanisme:

- **`ANALYZE` (manual)**
  Digunakan ketika kita ingin memperbarui statistik secara langsung

- **Auto Vacuum (otomatis)**
  Proses background yang secara berkala:
  - Membersihkan data yang sudah tidak terpakai
  - Sekaligus memperbarui statistik

#### Contoh Menjalankan ANALYZE

Jika kita ingin memastikan statistik selalu up-to-date, kita bisa menjalankan:

```sql id="p2x8rm"
-- Memperbarui statistik tabel secara manual
ANALYZE users;
```

Perintah ini akan membuat PostgreSQL:

- Mengambil sampel data dari tabel `users`
- Menghitung ulang distribusi dan metrik penting lainnya
- Menyimpan hasilnya untuk digunakan oleh query planner

#### Kenapa Statistik Ini Penting?

Karena semua keputusan query planner bergantung pada **estimasi**, bukan data aktual saat itu juga.

Jika statistik akurat:

- PostgreSQL bisa memilih strategi terbaik
- Query berjalan lebih cepat dan efisien

Jika statistik tidak akurat:

- Estimasi bisa meleset
- PostgreSQL bisa memilih strategi yang salah
  (misalnya tidak menggunakan _index_ padahal seharusnya bisa)

#### Kapan Perlu Menjalankan ANALYZE Manual?

Meskipun Auto Vacuum biasanya cukup, ada kondisi di mana kita perlu turun tangan:

- Setelah melakukan _insert_ dalam jumlah besar
- Setelah _update_ atau _delete_ massal
- Setelah perubahan data yang signifikan dalam waktu singkat

Dalam kasus seperti ini, statistik lama bisa menjadi tidak relevan, sehingga:

> Menjalankan `ANALYZE` secara manual membantu PostgreSQL “melihat ulang” kondisi data terbaru.

#### Cara Berpikir yang Tepat

Anggap statistik ini sebagai “peta” yang digunakan PostgreSQL untuk menavigasi data.

- Jika petanya akurat → perjalanan (query) efisien
- Jika petanya usang → bisa salah jalan

Dengan memahami peran statistik internal ini, kita jadi tahu bahwa performa query tidak hanya bergantung pada _index_, tetapi juga pada **seberapa akurat informasi yang dimiliki PostgreSQL tentang datanya**.

## 3. Hubungan Antar Konsep

Berikut adalah alur berpikir yang utuh saat mengevaluasi kandidat _index_:

```
Lihat Query yang Sering Dijalankan
          ↓
Identifikasi Kolom yang Difilter (WHERE clause)
          ↓
Hitung Cardinality → Berapa banyak nilai distinct?
          ↓
Hitung Selectivity → Seberapa besar rasionya terhadap total baris?
          ↓
Periksa Distribusi Data → Apakah data merata atau skewed?
          ↓
Evaluasi Selectivity per Nilai → Terutama jika data sangat skewed
          ↓
Keputusan: Buat Index / Tidak
```

Ketiga konsep — **cardinality**, **selectivity**, dan **distribusi data** — saling melengkapi. Tidak ada satu metrik tunggal yang cukup untuk membuat keputusan. Kamu harus mempertimbangkan ketiganya bersamaan dengan **pola query** yang aktual.

---

## 4. Kesimpulan

- **Cardinality** = jumlah nilai unik dalam kolom. Angka tinggi belum tentu berarti _index_ bagus.
- **Selectivity** = rasio nilai unik terhadap total baris. Semakin mendekati **1.0**, semakin baik _index_-nya.
- **Primary key** selalu memiliki selectivity sempurna (= 1.0).
- Kolom dengan selectivity rendah (seperti Boolean) **tetap bisa berguna** sebagai _index_ jika distribusi datanya sangat tidak merata dan query menargetkan nilai yang jarang muncul.
- Postgres menggunakan **statistik internal** (bukan query real-time) untuk memutuskan apakah _index_ digunakan. Statistik ini perlu diperbarui setelah perubahan data besar dengan `ANALYZE`.
- Tujuan utama sebuah _index_: **mempersempit hasil pencarian ke sesedikit mungkin baris, secepat mungkin**.
