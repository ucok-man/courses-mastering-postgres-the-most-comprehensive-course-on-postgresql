# 📝 Functional Indexes (Index pada Fungsi/Ekspresi) di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas konsep **functional indexes** (atau _expression indexes_) di PostgreSQL — yaitu kemampuan untuk membuat index bukan pada kolom biasa, melainkan pada **hasil dari sebuah fungsi atau ekspresi**. Teknik ini sangat berguna untuk mengoptimalkan query yang melibatkan transformasi data, seperti ekstraksi domain dari email, pencarian case-insensitive, atau pengambilan key dari JSON. Functional indexes merupakan peningkatan signifikan dari index biasa dan dapat mengangkat kemampuan database dari level pemula ke level menengah-lanjut.

---

## 2. Konsep Utama

### a. Apa Itu Functional Index?

- Selama ini, index biasanya dibuat pada satu kolom atau beberapa kolom sekaligus.
- **Functional index** adalah index yang dibuat pada **hasil evaluasi sebuah fungsi atau ekspresi**, bukan langsung pada kolom.
- Hasilnya disimpan di dalam **B-tree** (struktur data pohon yang digunakan PostgreSQL untuk index), sehingga saat query dijalankan, komputasi fungsi tidak perlu diulangi — cukup dicari di B-tree.
- Setiap kali sebuah baris di-_insert_, di-_update_, atau di-_delete_, fungsi tersebut dievaluasi ulang dan hasilnya diperbarui di dalam B-tree.

> **Analogi:** Bayangkan kamu punya buku telepon yang diurutkan bukan berdasarkan nama lengkap, tetapi berdasarkan inisial nama belakang. Itulah seperti functional index — kamu mengindeks _hasil transformasi_ dari data asli.

**Perbandingan dengan Generated Column:**
| | Generated Column | Functional Index |
|---|---|---|
| Cara kerja | Nilai fungsi disimpan sebagai kolom baru | Nilai fungsi disimpan di B-tree (index) |
| Terlihat di tabel? | Ya, tampil sebagai kolom | Tidak, tidak tampil sebagai kolom |
| Tujuan utama | Menyimpan & menampilkan nilai turunan | Mempercepat query/pencarian |

---

### b. Sintaks Pembuatan Functional Index

Perbedaan kunci dengan index biasa ada pada penggunaan **tanda kurung tambahan** di dalam definisi index:

```sql
-- Index biasa (pada kolom)
CREATE INDEX nama_index ON nama_tabel (nama_kolom);

-- Functional index (pada hasil fungsi)
CREATE INDEX nama_index ON nama_tabel ((fungsi(nama_kolom)));
--                                     ^^ kurung dalam ^^
```

> **Catatan penting:** Tanda kurung tambahan (double parentheses) adalah sinyal ke PostgreSQL bahwa kamu ingin mengindeks ekspresi/fungsi, bukan nama kolom biasa.

---

### c. Contoh 1 — Mengindeks Domain dari Alamat Email

**Skenario:** Tabel `users` memiliki kolom `email`. Kita ingin mencari pengguna berdasarkan domain email mereka (misalnya: `gmail.com`, `beer.com`).

**Fungsi yang digunakan:**

```sql
split_part(email, '@', 2)
-- Memecah string email menggunakan '@' sebagai delimiter,
-- lalu mengambil bagian ke-2 (yaitu domain).
```

**Contoh query tanpa index (lambat):**

```sql
SELECT * FROM users
WHERE split_part(email, '@', 2) = 'beer.com';
-- PostgreSQL harus mengevaluasi split_part() untuk SETIAP baris
```

**Membuat functional index:**

```sql
CREATE INDEX domain ON users ((split_part(email, '@', 2)));
```

**Query setelah index dibuat (cepat):**

```sql
SELECT * FROM users
WHERE split_part(email, '@', 2) = 'beer.com';
-- PostgreSQL langsung mencari di B-tree, tanpa menghitung ulang fungsi
```

Setelah index dibuat, PostgreSQL akan melakukan **Index Scan on domain** — artinya index berhasil digunakan.

> ⚠️ **Perhatian:** Query **harus menggunakan fungsi yang sama persis** seperti saat index dideklarasikan. Jika fungsinya berbeda (misalnya pakai `substring` alih-alih `split_part`), PostgreSQL tidak akan mengenali bahwa index itu relevan, dan akan kembali ke Sequential Scan.

---

### d. Contoh 2 — Case-Insensitive Search (Pencarian Tanpa Memperhatikan Huruf Kapital)

**Skenario:** Pengguna bisa mengetik email dengan huruf kapital sembarangan (`Aaron@Example.COM`), namun kita ingin tetap menemukannya di database.

**Membuat functional index dengan `lower()`:**

```sql
CREATE INDEX email_lower ON users ((lower(email)));
```

**Query yang BENAR (menggunakan index):**

```sql
SELECT * FROM users
WHERE lower(email) = lower('Aaron@Example.COM');
-- atau langsung:
WHERE lower(email) = 'aaron@example.com';
```

**Query yang SALAH (tidak menggunakan index):**

```sql
SELECT * FROM users
WHERE email = 'Aaron@Example.COM';
-- Index tidak dipakai karena query tidak menggunakan fungsi lower()
```

> 💡 **Penting:** Saat menggunakan functional index berbasis `lower()`, kamu **juga harus me-lowercase input pengguna** sebelum melakukan perbandingan. Jika tidak, hasilnya bisa kosong karena membandingkan string yang belum di-lowercase dengan nilai index yang sudah sepenuhnya lowercase.

---

### e. Use Case Lainnya yang Relevan

#### 1. Tidak perlu mengubah kode aplikasi

Functional index sangat berguna ketika:

- Kamu tidak memiliki kontrol penuh atas query yang dijalankan (misalnya dari library pihak ketiga, ORM, atau tool eksternal).
- Sistem eksternal sudah mengeluarkan query dengan fungsi tertentu.
- Kamu cukup **menambahkan index di sisi database** tanpa menyentuh kode aplikasi sama sekali.

#### 2. Mengindeks bagian dari JSON

Functional index sangat populer untuk mengekstrak key tertentu dari kolom bertipe JSON/JSONB:

```sql
-- Contoh: mengindeks field 'status' dari kolom JSON bernama 'metadata'
CREATE INDEX idx_status ON orders ((metadata->>'status'));

-- Query yang akan menggunakan index ini:
SELECT * FROM orders WHERE metadata->>'status' = 'shipped';
```

> Topik indexing JSON akan dibahas lebih mendalam di video terpisah.

---

## 3. Hubungan Antar Konsep

```
Data Asli (Kolom)
      │
      ▼
  Fungsi/Ekspresi (misal: lower(), split_part(), JSON extraction)
      │
      ├──► Generated Column → Nilai disimpan sebagai kolom baru (terlihat)
      │
      └──► Functional Index → Nilai disimpan di B-tree (tidak terlihat, tapi cepat)
```

- **Functional index** dan **generated column** adalah dua cara berbeda untuk "menyimpan" hasil komputasi fungsi.
- Keduanya bertujuan menghindari komputasi berulang saat query, namun dengan pendekatan berbeda.
- Functional index tidak mengubah struktur tabel, tidak menambah kolom baru, sehingga lebih transparan bagi aplikasi.
- Kunci keberhasilan functional index: **query harus menggunakan ekspresi yang identik** dengan yang digunakan saat index dibuat — PostgreSQL mencocokkan keduanya secara eksak.

---

## 4. Kesimpulan

Functional indexes adalah fitur PostgreSQL yang powerful namun sering diabaikan. Berikut poin-poin kunci:

- Index tidak hanya bisa dibuat pada kolom, tetapi juga pada **hasil fungsi atau ekspresi**.
- Hasilnya disimpan di **B-tree** sehingga pencarian menjadi sangat cepat tanpa harus menghitung ulang fungsi.
- Sintaks kuncinya: gunakan **tanda kurung ganda** saat mendefinisikan ekspresi dalam `CREATE INDEX`.
- Query **harus menggunakan ekspresi yang sama persis** agar PostgreSQL bisa mencocokkan index yang tepat.
- Kasus penggunaan yang paling umum: **case-insensitive search** (dengan `lower()`), **ekstraksi domain email** (dengan `split_part()`), dan **indexing field JSON**.
- Tidak perlu mengubah kode aplikasi — cukup tambahkan index di sisi database.
- Menguasai functional indexes dapat meningkatkan kemampuan kamu sebagai DBA dari level 101 ke level 201 bahkan 301.
