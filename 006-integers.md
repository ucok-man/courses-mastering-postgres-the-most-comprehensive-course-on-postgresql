# 📝 Tipe Data Integer di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tipe data integer di PostgreSQL, fokus pada bagaimana memilih tipe integer yang tepat untuk data dalam bounded range (rentang terbatas). Pembahasan mencakup tiga tipe utama: `SMALLINT`, `INTEGER` (atau `INT`), dan `BIGINT`, serta bagaimana memilih yang paling sesuai berdasarkan prinsip **representative** (mencakup seluruh range data) dan **compact** (hemat space tanpa mengorbankan data).

## 2. Konsep Utama

### a. Prinsip Memilih Tipe Data Integer

Saat menentukan tipe data integer di database, tujuan utamanya bukan sekadar “yang penting jalan”, tetapi memilih tipe yang **tepat secara makna**, **aman untuk masa depan**, dan **efisien dalam penyimpanan**. Ada dua prinsip utama yang perlu selalu diingat agar keputusan ini tidak menimbulkan masalah di kemudian hari.

#### 1. Representative (Representatif)

Prinsip representatif berarti tipe data yang dipilih **harus mampu mewakili seluruh kemungkinan nilai yang realistis**, bukan hanya data yang saat ini terlihat.

Beberapa hal penting yang perlu dipahami:

- Tipe data harus **mencakup seluruh range data yang mungkin terjadi**
- **Jangan pernah** memotong, membatasi, atau “menyetel paksa” data hanya demi menghemat beberapa byte
- Fokuslah pada **bounds of reality**, bukan sekadar data historis atau kondisi saat ini

Sebagai contoh, jika hari ini data umur pengguna hanya berkisar antara 0–90 tahun, bukan berarti itulah batas maksimal selamanya. Pertanyaannya bukan “berapa datanya sekarang?”, melainkan “berapa nilai maksimal yang masuk akal untuk terjadi?”.

#### 2. Compact (Ringkas)

Setelah memastikan data terwakili dengan aman, barulah kita berbicara soal efisiensi.

Prinsip compact menekankan bahwa:

- Jangan menggunakan tipe data yang terlalu besar untuk range yang kecil
- Semakin besar tipe data, semakin besar pula konsumsi storage dan memory
- Pilih **tipe terkecil yang masih aman secara representatif**

Contoh kesalahan umum adalah menggunakan `BIGINT` untuk data yang jelas-jelas hanya bernilai kecil, seperti status (0/1), level (1–10), atau umur manusia.

#### Ilustrasi Proses Pemilihan Tipe Data

Agar alurnya lebih jelas, berikut cara berpikir yang sistematis saat memilih tipe data integer.

```
Langkah 1: Analisis data Anda
├── Periksa range data saat ini (contoh: 0–90 tahun)
└── Pertanyaan: Apakah ini bounds of reality?

Langkah 2: Tentukan bounds of reality
├── Data saat ini: 0–90 tahun
├── Reality check: Apakah mungkin ada nilai ekstrem di masa depan?
└── Pilihan reasonable: misalnya 0–1000 untuk antisipasi

Langkah 3: Cocokkan dengan tipe data PostgreSQL
└── Pilih tipe integer terkecil yang bisa menampung range tersebut
```

Pendekatan ini membantu kita tidak terjebak pada kondisi sekarang saja, tetapi juga tidak berlebihan dalam memilih tipe data.

#### Contoh Kasus: Menyimpan Umur Pengguna

Mari kita lihat contoh konkret dalam bentuk SQL.

```sql
-- ❌ BURUK: Menggunakan BIGINT untuk umur (overkill)
CREATE TABLE users (
    name TEXT,
    age BIGINT  -- Range: sekitar -9 quintillion sampai +9 quintillion
);
```

Secara teknis, kode di atas memang valid dan tidak akan error. Namun secara desain, ini adalah pilihan yang buruk. Umur manusia tidak akan pernah mendekati batas maksimal `BIGINT`. Menggunakan tipe sebesar ini hanya membuang resource tanpa manfaat nyata.

Sekarang bandingkan dengan pendekatan yang lebih tepat:

```sql
-- ✅ BAIK: Menggunakan SMALLINT
CREATE TABLE users (
    name TEXT,
    age SMALLINT  -- Range: -32.768 sampai 32.767
);
```

`SMALLINT` sudah lebih dari cukup untuk menyimpan umur manusia, bahkan jika kita mengantisipasi nilai ekstrem sekalipun. Dengan pilihan ini:

- Data tetap aman dan representatif
- Storage lebih efisien
- Desain tabel mencerminkan makna data dengan lebih jelas

Intinya, memilih tipe data integer yang baik adalah soal **menyeimbangkan keamanan data dan efisiensi**, bukan sekadar memilih tipe terbesar agar “tidak repot”.

### b. Tiga Tipe Integer di PostgreSQL

PostgreSQL menyediakan tiga tipe data integer utama dengan ukuran dan kapasitas yang berbeda. Masing-masing dirancang untuk kebutuhan yang berbeda pula. Memahami perbedaan ini membantu kita memilih tipe data yang tepat sejak awal, sehingga desain database tetap efisien, aman, dan mudah berkembang.

#### 1. SMALLINT (alias: INT2)

`SMALLINT` adalah tipe integer paling kecil yang disediakan PostgreSQL. Tipe ini cocok untuk data numerik dengan rentang yang benar-benar terbatas.

##### Spesifikasi

- **Ukuran:** 2 bytes
- **Range nilai:** -32.768 sampai 32.767
- **Alias:** `INT2`, yang secara literal berarti “integer 2 bytes”

##### Kapan Menggunakan

Gunakan `SMALLINT` jika:

- Data memiliki range kecil dan jelas
- Nilai tidak mungkin melebihi ±32 ribu
- Contoh penggunaan: umur, persentase, rating, level, atau status numerik kecil

##### Contoh Penggunaan

```sql
CREATE TABLE users (
    name TEXT,
    age INT2  -- Alias dari SMALLINT
);
```

Pada tabel di atas, kolom `age` menggunakan `INT2`, yang sudah lebih dari cukup untuk menyimpan umur manusia.

```sql
-- Insert data normal
INSERT INTO users (name, age) VALUES
    ('Alice', 25),
    ('Bob', 67),
    ('Charlie', 0);  -- Bayi (0 tahun valid)
```

Semua data di atas akan tersimpan tanpa masalah karena masih berada dalam range `SMALLINT`.

```sql
-- ❌ Insert data di luar range
INSERT INTO users (name, age) VALUES ('Robot', 40000);
-- ERROR: smallint out of range
```

Ketika nilai `40000` dimasukkan, PostgreSQL langsung menolak data tersebut karena melewati batas maksimum `SMALLINT`.

##### Catatan Penting tentang Error

- Error `out of range` memberikan **perlindungan dasar terhadap integritas data**
- Namun ini **bukan pengganti business logic validation**
- Contohnya, umur `32000` tahun masih valid secara tipe data, tetapi tidak masuk akal secara bisnis
- Untuk aturan bisnis, gunakan **CHECK constraint** agar validasi lebih tepat dan eksplisit

#### 2. INTEGER (alias: INT, INT4)

`INTEGER` adalah tipe integer “tengah-tengah” dan merupakan pilihan paling umum di PostgreSQL.

##### Spesifikasi

- **Ukuran:** 4 bytes (dua kali `SMALLINT`)
- **Range nilai:** -2.147.483.648 sampai 2.147.483.647 (sekitar ±2,1 miliar)
- **Alias:** `INT` atau `INT4`

##### Kapan Menggunakan

Gunakan `INTEGER` jika:

- Membutuhkan range nilai yang cukup besar, tetapi tidak ekstrem
- Tidak yakin apakah `SMALLINT` cukup aman
- Menginginkan pilihan default yang aman dan fleksibel

Banyak framework dan ORM menggunakan `INTEGER` sebagai default saat membuat kolom numerik atau migration, karena tipe ini jarang menjadi bottleneck di tahap awal.

##### Contoh Penggunaan

```sql
CREATE TABLE warehouse_items (
    item_name TEXT,
    stock INT4  -- Bisa menampung hingga ±2.1 miliar unit
);
```

```sql
INSERT INTO warehouse_items (item_name, stock) VALUES
    ('Laptop', 1500),
    ('Mouse', 25000),
    ('Monitor', 8500000);  -- Jutaan unit masih aman
```

Semua nilai di atas jauh dari batas maksimum `INTEGER`, sehingga sangat aman untuk penggunaan jangka panjang.

##### Rekomendasi Penggunaan

- Untuk **kolom numerik biasa**: `INTEGER` biasanya sudah cukup
- Untuk **Primary Key ID**: lebih disarankan menggunakan `BIGINT`
- Jika benar-benar yakin range kecil: `SMALLINT` bisa menjadi pilihan yang lebih efisien

#### 3. BIGINT (alias: INT8)

`BIGINT` adalah tipe integer terbesar di PostgreSQL. Tipe ini dirancang untuk data yang skalanya bisa tumbuh sangat besar.

##### Spesifikasi

- **Ukuran:** 8 bytes
- **Range nilai:**
  -9.223.372.036.854.775.808 sampai
  9.223.372.036.854.775.807
- Kurang lebih ±9 **quintillion** (9 dengan 18 angka nol)
- **Alias:** `INT8`

##### Kapan Menggunakan

Gunakan `BIGINT` jika:

- Menyimpan **Primary Key ID** yang terus bertambah
- Data berpotensi berkembang sangat besar
- Ingin menghindari risiko kehabisan nilai di masa depan

Karena itu, `BIGINT` sangat direkomendasikan untuk kolom ID pada tabel-tabel utama.

##### Contoh Penggunaan

```sql
-- ✅ REKOMENDASI: Gunakan BIGINT untuk Primary Key
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    total_amount DECIMAL(10,2)
);
```

Dengan pendekatan ini, kita hampir tidak perlu khawatir kehabisan ID, bahkan untuk sistem berskala sangat besar.

```sql
-- Contoh: menyimpan ukuran file dalam bytes
CREATE TABLE files (
    file_name TEXT,
    file_size_bytes BIGINT
);
```

```sql
INSERT INTO files (file_name, file_size_bytes) VALUES
    ('small.txt', 1024),
    ('large_video.mp4', 5368709120);  -- 5 GB dalam bytes
```

Nilai ukuran file dengan satuan byte bisa dengan mudah melampaui batas `INTEGER`, sehingga `BIGINT` adalah pilihan yang tepat.

#### Perbandingan Ukuran dan Dampaknya

```
SMALLINT: 2 bytes  [========]
INTEGER:  4 bytes  [================]
BIGINT:   8 bytes  [================================]
```

Perbedaan ukuran ini akan sangat terasa pada tabel berskala besar. Sebagai gambaran kasar:

- 1 miliar baris dengan `SMALLINT` ≈ 2 GB
- 1 miliar baris dengan `INTEGER` ≈ 4 GB
- 1 miliar baris dengan `BIGINT` ≈ 8 GB

Karena itu, pemilihan tipe integer yang tepat bisa berdampak besar pada penggunaan storage dan performa, terutama ketika sistem tumbuh menjadi sangat besar.

### c. Unsigned Integer di PostgreSQL

Saat bekerja dengan PostgreSQL, ada satu hal penting yang sering mengejutkan developer—terutama yang sebelumnya terbiasa dengan MySQL—yaitu **tidak adanya tipe unsigned integer**. Memahami keputusan desain ini dan cara menyiasatinya sangat penting agar skema database tetap aman dan konsisten.

#### Fakta Penting

Beberapa fakta dasar yang perlu dipahami sejak awal:

- PostgreSQL **tidak menyediakan** tipe data `UNSIGNED INTEGER`
- Ini berbeda dengan MySQL, yang memiliki variasi seperti `UNSIGNED INT`, `UNSIGNED BIGINT`, dan lain-lain
- Semua tipe integer di PostgreSQL bersifat **signed**, artinya selalu memiliki kemungkinan nilai negatif

Dengan kata lain, di PostgreSQL tidak ada cara bawaan untuk “mematikan” nilai negatif hanya dengan memilih tipe data tertentu.

#### Perbandingan dengan Database Lain

Perbedaan ini akan terlihat jelas jika kita bandingkan langsung dengan MySQL.

```sql
-- MySQL (memiliki UNSIGNED)
CREATE TABLE items (
    quantity INT UNSIGNED  -- Range: 0 sampai 4.294.967.295
);
```

Di MySQL, `INT UNSIGNED` menggandakan kapasitas sisi positif dengan menghilangkan nilai negatif. Tipe ini sering dipakai untuk stok, jumlah, atau counter yang secara logika tidak pernah bernilai minus.

Sekarang bandingkan dengan PostgreSQL:

```sql
-- PostgreSQL (TIDAK punya UNSIGNED)
CREATE TABLE items (
    quantity INTEGER  -- Range: -2,1 miliar sampai +2,1 miliar
);
```

Di sini, `INTEGER` selalu memiliki rentang negatif dan positif. PostgreSQL sengaja memilih pendekatan ini agar tipe data tetap konsisten dan sederhana, serta mendorong validasi dilakukan di level yang lebih tepat.

#### Solusi untuk Memaksa Nilai Positif

Karena tidak ada `UNSIGNED`, PostgreSQL menyediakan mekanisme yang lebih fleksibel dan eksplisit, yaitu **CHECK constraint**. Dengan constraint ini, kita bisa mendefinisikan aturan bisnis secara jelas.

```sql
-- Memaksa quantity tidak boleh negatif
CREATE TABLE products (
    product_name TEXT,
    quantity INTEGER CHECK (quantity >= 0)
);
```

Dengan definisi ini, PostgreSQL akan menolak setiap data yang melanggar aturan `quantity >= 0`.

```sql
-- Contoh lain: validasi umur
CREATE TABLE users (
    name TEXT,
    age SMALLINT CHECK (age >= 0 AND age <= 150)
);
```

Pada contoh umur, constraint tidak hanya mencegah nilai negatif, tetapi juga membatasi nilai maksimum agar tetap masuk akal secara bisnis.

```sql
-- Pengujian constraint
INSERT INTO products (product_name, quantity) VALUES ('Laptop', -5);
-- ERROR: new row violates check constraint
```

Saat nilai `-5` dimasukkan, PostgreSQL langsung menolak data tersebut. Ini menunjukkan bahwa validasi dilakukan secara konsisten di level database, bukan hanya bergantung pada aplikasi.

#### Catatan Penting

Ada beberapa poin tambahan yang perlu diperhatikan:

- SQLite juga **tidak memiliki** konsep unsigned integer, sehingga pendekatannya mirip dengan PostgreSQL
- Developer MySQL perlu menyesuaikan pola pikir saat migrasi ke PostgreSQL
- `CHECK constraint` jauh lebih **powerful dan ekspresif** dibandingkan sekadar `UNSIGNED`, karena bisa memodelkan aturan bisnis secara eksplisit

Dengan pendekatan ini, PostgreSQL mendorong desain skema yang lebih jelas, aman, dan mudah dipahami, terutama untuk sistem yang kompleks dan terus berkembang.

### d. Penggunaan Framework untuk Migrations

Dalam praktik sehari-hari, penggunaan framework dan ORM untuk database migrations adalah hal yang sangat umum. Framework seperti Django ORM, Rails migrations, TypeORM, dan sejenisnya memang dirancang untuk mempermudah pekerjaan developer. Namun, kemudahan ini tetap harus dibarengi dengan pemahaman dan tanggung jawab teknis.

#### Perspektif tentang Framework

Pertama, penting untuk meluruskan satu hal: **menggunakan framework bukanlah sebuah kesalahan**.

Framework membantu kita:

- Membuat dan mengelola migration dengan lebih cepat
- Menjaga konsistensi skema database antar environment
- Mengurangi pekerjaan manual yang berulang

Karena itu, **tidak ada masalah sama sekali** menggunakan ORM atau migration tool. Anda juga **bukan “developer palsu”** hanya karena tidak menulis SQL secara manual setiap hari.

#### Tanggung Jawab Developer Tetap Ada

Meskipun framework melakukan banyak hal secara otomatis, tanggung jawab tetap ada di tangan developer.

Beberapa hal yang **wajib** dipahami:

- Anda harus **mengerti SQL** yang dihasilkan oleh framework
- Migration file bukan “black box”, melainkan sesuatu yang perlu **direview**
- Anda harus memahami **implikasi pilihan tipe data**, baik dari sisi range, storage, maupun performa

Framework hanya membantu men-generate kode. Keputusan desain tetap Anda yang menentukan kualitas akhirnya.

#### Contoh Masalah Umum pada Framework

Masalah yang sering terjadi adalah penggunaan **default behavior** tanpa evaluasi.

```javascript
// TypeORM - Default ke INTEGER untuk semua numeric
@Entity()
class User {
  @PrimaryGeneratedColumn()
  id: number; // Generated: INTEGER (bukan BIGINT!)

  @Column()
  age: number; // Generated: INTEGER (boros untuk umur!)
}
```

Sekilas, kode ini terlihat aman dan sederhana. Namun jika diperhatikan lebih dalam:

- `id` sebagai primary key menggunakan `INTEGER`, yang berpotensi kehabisan nilai pada sistem berskala besar
- `age` juga menggunakan `INTEGER`, padahal range umur sangat kecil dan lebih cocok menggunakan `SMALLINT`

Masalah ini tidak muncul karena framework “buruk”, tetapi karena default-nya tidak selalu sesuai dengan kebutuhan domain kita.

#### Best Practice dalam Menggunakan Framework

Solusi terbaik adalah **bersikap eksplisit** terhadap tipe data yang digunakan.

```javascript
// Menentukan tipe data secara eksplisit
@Entity()
class User {
  @PrimaryGeneratedColumn("bigint") // ✅ Explicit BIGINT untuk ID
  id: number;

  @Column({ type: "smallint" }) // ✅ Explicit SMALLINT untuk umur
  age: number;
}
```

Dengan pendekatan ini:

- Skema database menjadi lebih jelas dan terkontrol
- Tidak bergantung pada asumsi default framework
- Risiko masalah jangka panjang bisa ditekan sejak awal

Intinya, framework adalah **alat bantu**, bukan pengganti pemahaman. Gunakan framework untuk efisiensi, tetapi tetap pahami apa yang terjadi di balik layar agar desain database tetap solid dan profesional.

### e. Primary Key: Selalu Gunakan BIGINT

Untuk desain database jangka panjang, ada satu rekomendasi yang sangat kuat dan konsisten: **gunakan `BIGINT` sebagai Primary Key secara default**. Keputusan ini mungkin terlihat berlebihan di awal, tetapi justru sangat masuk akal jika dilihat dari perspektif skalabilitas dan risiko di masa depan.

#### Rekomendasi Kuat

Berikut contoh implementasi yang direkomendasikan di PostgreSQL.

```sql
-- ✅ BEST PRACTICE: Default ke BIGINT untuk Primary Key
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,  -- Auto-increment berbasis BIGINT
    username VARCHAR(50)
);
```

`BIGSERIAL` secara otomatis membuat kolom `BIGINT` dengan sequence sebagai auto-increment, sehingga sangat praktis untuk primary key.

Alternatif lain, jika auto-increment tidak digunakan:

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name TEXT
);
```

Pendekatan ini cocok jika ID di-generate dari aplikasi atau sistem lain, tetapi tetap mempertahankan kapasitas besar dari `BIGINT`.

#### Alasan Menggunakan BIGINT untuk Primary Key

Keputusan menggunakan `BIGINT` bukan tanpa alasan. Ada beberapa pertimbangan penting yang membuatnya menjadi pilihan paling aman.

##### 1. Tidak Ada Downside yang Signifikan

- `BIGINT` hanya menggunakan **8 bytes per baris**, yang relatif kecil untuk kolom primary key
- Primary key hampir selalu di-index, sehingga perbedaan ukuran ini **tidak berdampak besar pada performa**
- Dibandingkan risiko yang dihindari, biaya storage ini sangat kecil

##### 2. Menghindari Bencana di Masa Depan

- Batas maksimum `INTEGER` hanya sekitar **2,1 miliar**
- Angka ini terdengar besar, tetapi bisa tercapai pada aplikasi yang:

  - Viral
  - Berumur panjang
  - Menghasilkan data secara masif

- Migrasi dari `INTEGER` ke `BIGINT` pada primary key di production:

  - Sangat kompleks
  - Berisiko
  - Sering membutuhkan downtime

Dengan kata lain, memilih `INTEGER` untuk PK adalah menunda masalah besar ke masa depan.

##### 3. Peace of Mind

- `BIGINT` mampu menampung hingga **9 quintillion** nilai
- Secara praktis, ini berarti “hampir tidak terbatas”
- Anda tidak perlu memikirkan ulang desain primary key saat sistem tumbuh

Keputusan ini memberikan ketenangan jangka panjang, terutama untuk sistem yang diharapkan terus berkembang.

#### Kisah Horor (Ilustrasi Hipotetis)

Untuk menggambarkan risikonya, bayangkan skenario berikut.

```sql
-- Tahun 2020: Aplikasi baru
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,  -- INTEGER (maksimum ±2.1 miliar)
    content TEXT
);
```

Di tahap awal, desain ini tampak baik-baik saja. Aplikasi berjalan normal tanpa masalah.

```sql
-- Tahun 2024: Aplikasi viral, mendekati 2 miliar posts
INSERT INTO posts (content) VALUES ('New post');
-- ERROR: integer out of range
```

Ketika jumlah data mendekati batas maksimum `INTEGER`, sistem tiba-tiba gagal menambahkan data baru. Pada titik ini, solusinya sangat tidak menyenangkan.

- Harus melakukan migrasi ke `BIGINT`
- Membutuhkan downtime atau strategi migrasi yang rumit
- Semua foreign key yang mengacu ke `posts.id` harus ikut diubah
- Risiko error dan data corruption meningkat drastis

Masalah ini sebenarnya bisa dihindari sepenuhnya dengan satu keputusan sederhana di awal: **menggunakan `BIGINT` sebagai primary key sejak awal desain**.

## 3. Hubungan Antar Konsep

**Alur pemikiran dalam memilih tipe integer:**

```
1. Analisis Data
   ├── Apa bounds dari data saat ini?
   ├── Apa bounds of reality?
   └── Apakah data bisa berkembang drastis?

2. Evaluasi Tipe Data
   ├── SMALLINT: Untuk data kecil & stabil (umur, rating)
   ├── INTEGER: Default untuk kebanyakan kasus
   └── BIGINT: Untuk Primary Keys & data unlimited growth

3. Pertimbangan Tambahan
   ├── Apakah butuh unsigned? → Gunakan CHECK constraint
   ├── Business logic validation? → CHECK constraint / domain
   └── Framework default? → Review & customize bila perlu

4. Trade-off
   ├── Space vs Safety
   ├── SMALLINT hemat 6 bytes vs INTEGER (per row)
   └── Tapi jangan sacrifice data integrity!
```

**Hubungan dengan konsep lain:**

- **Check Constraints:** Melengkapi tipe data untuk business logic (dibahas video mendatang)
- **Domains:** Custom types untuk reusable constraints (dibahas video mendatang)
- **Indexing:** Tipe data lebih kecil = index lebih efisien
- **Table Compaction:** Column ordering untuk optimasi storage (dibahas jauh ke depan)

**Progression pemahaman:**

1. Pahami tipe data dasar (video ini)
2. Tambahkan constraints untuk validation
3. Optimize dengan indexing
4. Fine-tune dengan table structure optimization

## 4. Kesimpulan

Memilih tipe data integer yang tepat di PostgreSQL adalah tentang **balance** antara efisiensi storage dan safety data. Ada tiga pilihan utama:

- **SMALLINT (2 bytes):** Untuk data dengan range kecil dan stabil
- **INTEGER (4 bytes):** Default pilihan untuk kebanyakan kasus
- **BIGINT (8 bytes):** **Wajib** untuk Primary Key IDs, opsional untuk data unlimited growth

**Prinsip emas:**

1. **Analisis bounds of reality**, bukan hanya data saat ini
2. **Jangan sacrifice data integrity** demi hemat bytes
3. **Selalu gunakan BIGINT untuk Primary Keys**
4. **Gunakan CHECK constraint** untuk validation (bukan mengandalkan unsigned)
5. **Review generated SQL** dari framework - Anda tetap yang bertanggung jawab

PostgreSQL tidak memiliki unsigned integer seperti MySQL, tetapi CHECK constraints memberikan flexibility yang lebih powerful untuk business logic validation.

Pilihan tipe data yang tepat sejak awal akan mempermudah indexing, meningkatkan performa query, dan menghindari painful migration di masa depan. Ini adalah **investasi yang akan terbayar** seiring aplikasi berkembang.
