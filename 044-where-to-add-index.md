# 📝 Di Mana Harus Menambahkan Index di PostgreSQL?

## 1. Ringkasan Singkat

Video ini membahas strategi praktis dalam menentukan kolom mana yang perlu diberi index pada database PostgreSQL. Materi dimulai dari filosofi dasar indexing (seni vs sains), dilanjutkan dengan eksperimen langsung menggunakan tabel `users` berisi hampir satu juta baris untuk membuktikan kapan B-tree index benar-benar berguna. Topik yang dibuktikan mencakup _strict equality_, _range query_ (terbatas maupun tak terbatas), _ordering_, dan _grouping_.

---

## 2. Konsep Utama

### a. Indexing Adalah Seni, Bukan Sains

#### Memahami Perbedaan antara Schema dan Index

Saat kamu merancang schema database, pendekatannya cenderung ilmiah. Kamu bisa mulai dari menganalisis kebutuhan data, melihat jenis nilai yang akan disimpan, lalu memilih tipe data yang paling sesuai. Misalnya, kamu tahu sebuah kolom hanya berisi angka kecil, maka kamu bisa memilih tipe `SMALLINT`. Atau jika menyimpan teks panjang, kamu bisa gunakan `TEXT`. Semua ini bisa ditentukan secara logis dan cukup “pasti”.

Namun, ketika masuk ke dunia indexing, pendekatannya berubah.

Index tidak bisa ditentukan hanya dari melihat struktur tabel atau jenis data saja. Dua tabel dengan schema yang sama bisa saja membutuhkan index yang berbeda, tergantung bagaimana tabel tersebut digunakan oleh aplikasi.

#### Kenapa Index Disebut “Seni”?

Index disebut sebagai “seni” karena banyak melibatkan pertimbangan praktis dan pengalaman, bukan sekadar aturan baku.

Kamu tidak bisa hanya melihat tabel seperti ini:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT,
    created_at TIMESTAMP
);
```

Lalu langsung tahu index apa saja yang dibutuhkan. Kenapa? Karena yang paling penting bukan struktur tabelnya, melainkan bagaimana tabel ini akan di-query.

#### Peran Access Pattern

Hal utama yang harus kamu perhatikan adalah **access pattern**, yaitu pola bagaimana aplikasi mengakses data.

Contohnya:

- Jika aplikasi sering mencari user berdasarkan email:

  ```sql
  SELECT * FROM users WHERE email = 'user@example.com';
  ```

  Maka index pada kolom `email` sangat penting:

  ```sql
  CREATE INDEX idx_users_email ON users(email);
  ```

- Jika aplikasi sering menampilkan data terbaru:

  ```sql
  SELECT * FROM users ORDER BY created_at DESC LIMIT 10;
  ```

  Maka index pada `created_at` akan membantu:

  ```sql
  CREATE INDEX idx_users_created_at ON users(created_at DESC);
  ```

Perhatikan bahwa kedua index tersebut muncul bukan karena struktur tabelnya, tapi karena kebutuhan query-nya.

#### Intinya

Merancang index bukan soal “apa yang ada di tabel”, tapi lebih ke “bagaimana tabel itu digunakan”.

Karena itu:

- Kamu perlu memahami query yang sering dijalankan
- Mengamati performa
- Kadang mencoba beberapa pendekatan
- Dan belajar dari pengalaman

Di sinilah letak “seninya” — tidak selalu ada jawaban tunggal yang benar, tapi ada pendekatan yang lebih optimal tergantung konteks penggunaan.

### b. Jangan Memberi Index pada Setiap Kolom

#### Kesalahan Umum: “Semua Kolom Harus Di-Index”

Banyak developer berpikir bahwa karena index bisa mempercepat query, maka solusi terbaik adalah memasang index di setiap kolom. Sekilas terdengar masuk akal, tapi dalam praktiknya ini justru bisa menjadi masalah serius.

Index memang membantu proses pencarian, tapi bukan berarti semakin banyak index akan selalu semakin baik.

#### Index Itu Bukan Gratis

Perlu dipahami bahwa index adalah **struktur data tambahan** yang disimpan terpisah dari tabel utama. Artinya, setiap kali kamu membuat index pada sebuah kolom, database akan menyimpan salinan data tertentu dalam format yang dioptimalkan untuk pencarian.

Jika kamu membuat index di semua kolom, secara tidak langsung kamu sedang:

- Menggandakan data berkali-kali
- Menggunakan lebih banyak ruang penyimpanan
- Menambah beban kerja database

#### Dampak pada Operasi Write (INSERT, UPDATE, DELETE)

Setiap kali ada perubahan data, database tidak hanya mengubah data di tabel utama, tapi juga harus memperbarui semua index yang terkait.

Contohnya:

```sql
INSERT INTO users (email, created_at)
VALUES ('user@example.com', NOW());
```

Jika tabel `users` memiliki banyak index (misalnya di `email`, `created_at`, dan kolom lain), maka setiap `INSERT` akan memicu update ke semua index tersebut.

Akibatnya:

- `INSERT` menjadi lebih lambat
- `UPDATE` juga ikut berat, terutama jika kolom yang diubah memiliki index
- `DELETE` harus menghapus data dari semua index terkait

Semakin banyak index, semakin besar overhead-nya.

#### Performa SELECT Tidak Selalu Meningkat

Yang sering mengejutkan: banyak index tidak selalu membuat query `SELECT` lebih cepat.

Misalnya kamu punya query seperti ini:

```sql
SELECT *
FROM orders
WHERE col_a = 10 AND col_b = 20 AND col_c = 30;
```

Pendekatan yang kurang optimal:

```sql
CREATE INDEX idx_a ON orders(col_a);
CREATE INDEX idx_b ON orders(col_b);
CREATE INDEX idx_c ON orders(col_c);
```

Database mungkin hanya menggunakan salah satu index, atau harus menggabungkan beberapa index (yang tidak selalu efisien).

Pendekatan yang lebih baik:

```sql
CREATE INDEX idx_abc ON orders(col_a, col_b, col_c);
```

Dengan **composite index** seperti ini, database bisa langsung menemukan data yang cocok tanpa harus “menggabungkan” hasil dari beberapa index terpisah.

#### Ilustrasi Sederhana

Bayangkan kamu punya tiga index terpisah:

```
❌ index_col_a, index_col_b, index_col_c
```

Ini berarti database harus bekerja lebih keras, baik saat menulis data maupun saat mencoba menggabungkan hasil pencarian.

Bandingkan dengan satu index gabungan:

```
✅ index_(col_a, col_b, col_c)
```

Ini lebih efisien karena:

- Data tersusun sesuai kebutuhan query
- Lebih sedikit struktur yang harus dikelola
- Lebih optimal untuk query yang melibatkan beberapa kolom sekaligus

#### Intinya

Index memang powerful, tapi harus digunakan dengan bijak.

Daripada meng-index semua kolom:

- Fokus pada query yang sering dijalankan
- Buat index yang sesuai dengan pola akses data
- Pertimbangkan penggunaan composite index jika query melibatkan beberapa kolom

Dengan pendekatan ini, kamu tidak hanya mendapatkan performa yang lebih baik, tapi juga menjaga efisiensi resource database secara keseluruhan.

### c. Aturan "WHERE = Index" Tidak Cukup

#### Kenapa Aturan Ini Terlihat Benar

Ada aturan yang cukup populer: _“kalau kolom muncul di `WHERE`, berarti harus di-index.”_
Secara dasar, ini memang masuk akal.

Contohnya:

```sql
SELECT *
FROM users
WHERE email = 'user@example.com';
```

Karena database harus mencari data berdasarkan `email`, maka index pada kolom tersebut jelas membantu:

```sql
CREATE INDEX idx_users_email ON users(email);
```

Dengan index ini, database tidak perlu melakukan scan seluruh tabel, melainkan bisa langsung “lompat” ke data yang dicari.

#### Tapi Ini Baru Sebagian Cerita

Masalahnya, query di dunia nyata hampir tidak pernah sesederhana itu.

Jika kamu hanya fokus pada `WHERE`, kamu bisa melewatkan peluang optimasi yang jauh lebih besar. Bahkan, dalam beberapa kasus, index yang hanya berdasarkan `WHERE` saja tidak cukup optimal.

Seorang profesional tidak melihat query secara parsial, tapi sebagai satu kesatuan.

#### Melihat Query Secara Utuh

Mari kita lihat contoh query yang lebih realistis:

```sql
SELECT id, email, created_at
FROM users
WHERE status = 'active'
ORDER BY created_at DESC;
```

Kalau hanya mengikuti aturan “WHERE = index”, kamu mungkin akan membuat:

```sql
CREATE INDEX idx_users_status ON users(status);
```

Index ini memang membantu filtering. Tapi setelah itu, database masih harus melakukan sorting untuk `ORDER BY created_at DESC`, yang bisa mahal jika datanya besar.

Pendekatan yang lebih baik adalah mempertimbangkan kedua aspek sekaligus:

```sql
CREATE INDEX idx_users_status_created_at
ON users(status, created_at DESC);
```

Dengan index ini:

- Filtering berdasarkan `status` jadi cepat
- Data sudah tersusun sesuai `created_at DESC`, sehingga tidak perlu sorting tambahan

#### Jangan Lupakan Bagian Lain dari Query

Selain `WHERE` dan `ORDER BY`, ada beberapa bagian lain yang juga sangat memengaruhi desain index:

- **`GROUP BY`**
  Jika kamu sering melakukan agregasi:

  ```sql
  SELECT status, COUNT(*)
  FROM users
  GROUP BY status;
  ```

  Maka index pada `status` bisa membantu proses grouping.

- **`JOIN`**
  Untuk query yang menggabungkan tabel:

  ```sql
  SELECT u.email, o.total
  FROM users u
  JOIN orders o ON u.id = o.user_id;
  ```

  Index pada kolom join seperti `users.id` dan `orders.user_id` sangat penting agar proses join tidak menjadi lambat.

- **`SELECT` (Covering Index)**
  Kadang, kamu bisa membuat index yang juga mencakup kolom yang diambil:

  ```sql
  CREATE INDEX idx_users_status_created_at_email
  ON users(status, created_at DESC, email);
  ```

  Ini memungkinkan database mengambil semua data langsung dari index tanpa perlu mengakses tabel utama (disebut _covering index_).

#### Intinya

Aturan “WHERE = index” adalah titik awal yang baik, tapi tidak cukup untuk menghasilkan performa terbaik.

Untuk merancang index yang optimal:

- Lihat seluruh query, bukan hanya `WHERE`
- Perhatikan bagaimana data difilter, diurutkan, dikelompokkan, dan digabungkan
- Sesuaikan struktur index dengan pola akses tersebut

Dengan cara ini, kamu tidak hanya membuat query “jalan”, tapi benar-benar efisien dan scalable.

### d. Membuktikan Manfaat Index Secara Praktis

Untuk benar-benar memahami manfaat index, kita tidak cukup hanya teori — kita perlu melihat bagaimana database benar-benar mengeksekusi query.

Pada eksperimen ini, digunakan tabel `users` dengan sekitar **989.908 baris** dan beberapa kolom seperti `first_name`, `last_name`, `email`, `birthday`, `is_pro`, `created_at`, dan `updated_at`.

#### Setup Index

Kita mulai dengan membuat index pada kolom `birthday`:

```sql
CREATE INDEX b_day ON users USING BTREE (birthday);
```

Lalu, untuk melihat bagaimana PostgreSQL mengeksekusi query, kita gunakan perintah:

```sql
EXPLAIN SELECT * FROM users WHERE birthday = '1989-02-14';
```

`EXPLAIN` tidak menjalankan query, tetapi menunjukkan **rencana eksekusi** (execution plan) yang akan dipilih oleh database. Dari sinilah kita bisa melihat apakah index digunakan atau tidak.

#### d.1 — Strict Equality (Kesetaraan Tepat)

Kasus pertama adalah pencarian dengan nilai yang **persis sama**.

```sql
-- Tanpa index → Seq Scan (lambat)
EXPLAIN SELECT * FROM users WHERE first_name = 'Aaron';

-- Dengan index di birthday → Bitmap Index Scan (cepat)
EXPLAIN SELECT * FROM users WHERE birthday = '1989-02-14';
```

Perbedaannya cukup jelas:

- Query tanpa index akan menggunakan **Sequential Scan**, artinya PostgreSQL membaca seluruh tabel dari awal sampai akhir.
- Query dengan index menggunakan **Bitmap Index Scan**, yaitu mencari posisi data melalui index, lalu mengambil baris yang relevan saja.

Kesimpulannya:
Index tipe B-tree sangat efektif untuk pencarian dengan kondisi `=` (nilai yang sama persis).

#### d.2 — Unbounded Range (Rentang Tak Terbatas)

Sekarang kita coba query dengan kondisi rentang satu sisi:

```sql
-- "Lahir sebelum 14 Feb 1989" → Index DIGUNAKAN
EXPLAIN SELECT * FROM users WHERE birthday < '1989-02-14';

-- "Lahir setelah 14 Feb 1989" → Index TIDAK DIGUNAKAN
EXPLAIN SELECT * FROM users WHERE birthday > '1989-02-14';
```

Hasilnya mungkin terasa aneh di awal:

- Untuk kondisi `<`, index digunakan
- Untuk kondisi `>`, justru tidak digunakan

Kenapa bisa begitu?

Ini berkaitan dengan konsep **selectivity** — seberapa banyak data yang memenuhi kondisi tersebut.

Jika kondisi `birthday > '1989-02-14'` ternyata mencakup sebagian besar tabel, maka menggunakan index justru tidak efisien. Dalam kasus seperti ini, PostgreSQL memilih **Sequential Scan** karena membaca langsung seluruh tabel lebih cepat daripada bolak-balik melalui index.

#### d.3 — Bounded Range (Rentang Terbatas / Dua Sisi)

Sekarang kita batasi rentangnya dari dua sisi:

```sql
-- Semua user yang lahir di tahun 1989
EXPLAIN SELECT * FROM users
WHERE birthday >= '1989-01-01' AND birthday < '1990-01-01';
```

Di sini, index kembali digunakan.

Kenapa?

Karena rentangnya lebih spesifik, sehingga jumlah baris yang diambil lebih sedikit. Ini membuat penggunaan index menjadi efisien.

Biasanya, pada output `EXPLAIN`, kamu akan melihat bahwa kondisi index mencakup seluruh rentang tahun 1989.

#### d.4 — ORDER BY (Pengurutan)

Sekarang kita lihat bagaimana index memengaruhi pengurutan data:

```sql
-- Dengan index → Index Scan (data sudah terurut)
EXPLAIN SELECT * FROM users ORDER BY birthday;

-- Tanpa index → Seq Scan + Sort
EXPLAIN SELECT * FROM users ORDER BY first_name;
```

Perbedaannya sangat signifikan:

- Jika ada index pada kolom yang digunakan untuk `ORDER BY`, PostgreSQL bisa langsung membaca data dalam urutan yang sudah tersusun (**Index Scan**).
- Jika tidak ada index, PostgreSQL harus:
  1. Membaca seluruh tabel (Sequential Scan)
  2. Melakukan proses sorting tambahan

Proses sorting ini bisa sangat mahal, terutama untuk tabel besar.

#### d.5 — GROUP BY (Pengelompokan)

Terakhir, kita lihat efek index pada operasi agregasi:

```sql
EXPLAIN SELECT COUNT(*), birthday
FROM users
GROUP BY birthday;
```

Hasilnya menunjukkan bahwa PostgreSQL menggunakan **Parallel Index Scan** pada index `b_day`.

Artinya:

- Data dibaca melalui index
- Proses dilakukan secara paralel (multi-core)
- Tidak perlu membaca seluruh tabel secara linear

Ini menunjukkan bahwa index juga bisa memberikan peningkatan performa yang signifikan pada operasi `GROUP BY`, bukan hanya pada `WHERE`.

#### Intinya

Dari serangkaian eksperimen ini, kita bisa melihat bahwa manfaat index sangat nyata, tetapi penggunaannya bergantung pada jenis query:

- Sangat efektif untuk `=` (equality)
- Efektif untuk rentang tertentu (tergantung selectivity)
- Membantu `ORDER BY` tanpa perlu sorting tambahan
- Mempercepat `GROUP BY` dengan akses data yang lebih terstruktur

Dan yang paling penting:
Database tidak selalu menggunakan index — ia akan memilih strategi terbaik berdasarkan kondisi data dan query.

## 3. Hubungan Antar Konsep

Pemahaman tentang kapan harus menambahkan index dibangun secara bertahap:

1. **Filosofi dasar** → Indexing bukan tentang schema, melainkan tentang _access pattern_. Kamu harus tahu bagaimana data akan diakses sebelum memutuskan index apa yang dibutuhkan.

2. **Hindari index berlebihan** → Setiap index punya biaya: memperlambat operasi tulis dan mengonsumsi storage. Menambahkan index pada semua kolom justru kontraproduktif.

3. **Seluruh query sebagai panduan** → `WHERE` bukan satu-satunya pertimbangan. `ORDER BY`, `GROUP BY`, dan `JOIN` juga bisa dioptimalkan dengan index yang tepat.

4. **Pembuktian empiris** → Gunakan `EXPLAIN` untuk memverifikasi apakah index benar-benar digunakan oleh query planner PostgreSQL. Jangan hanya berasumsi.

5. **Selectivity** → Tidak semua kondisi akan memanfaatkan index. PostgreSQL cerdas: jika ia menilai _sequential scan_ lebih cepat (misalnya karena terlalu banyak baris cocok), ia akan memilih itu. Ini menunjukkan bahwa desain index harus mempertimbangkan distribusi data.

---

## 4. Kesimpulan

- Indexing adalah **seni** yang didorong oleh _access pattern_ query, bukan oleh struktur data atau schema semata.
- **Jangan** memberi index pada setiap kolom — ini boros dan kontraproduktif.
- Pertimbangkan **seluruh query** saat merancang strategi indexing: `WHERE`, `ORDER BY`, `GROUP BY`, dan `JOIN`.
- B-tree index terbukti efektif untuk: **strict equality**, **bounded range**, **unbounded range** (dengan caveat selectivity), **ordering**, dan **grouping**.
- Gunakan `EXPLAIN` untuk membuktikan secara empiris apakah index digunakan oleh PostgreSQL.
- Konsep **index selectivity** dan **composite index** akan dibahas lebih dalam di video-video berikutnya.
