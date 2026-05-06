# 📝 Duplicate Indexes dalam Database

## 1. Ringkasan Singkat

Video ini membahas masalah **duplicate index** (index ganda) dalam database — situasi di mana dua index yang berbeda secara teknis sebenarnya berfungsi sama dan salah satunya menjadi tidak berguna. Materi ini menjelaskan bagaimana duplicate index bisa terjadi tanpa disadari, mengapa hal itu merugikan performa database, dan bagaimana cara menghindarinya dengan memahami cara kerja B-tree index.

---

## 2. Konsep Utama

### a. Prinsip Dasar Indexing: Seimbang antara Kebutuhan dan Efisiensi

- Setiap index adalah **struktur data terpisah** yang harus dipelihara setiap kali terjadi operasi `INSERT`, `UPDATE`, atau `DELETE` — artinya setiap operasi tersebut juga harus menyentuh B-tree dari index terkait.
- Aturan praktisnya: **buat index sebanyak yang dibutuhkan, tapi sesedikit mungkin yang masih cukup**.
- Index yang tidak perlu hanya menambah beban pemeliharaan dan bisa membingungkan query planner, tim developer, dan diri sendiri.

**Ilustrasi biaya index:**

```
Operasi: INSERT INTO users (email, is_pro) VALUES ('a@b.com', true)

→ Update tabel utama
→ Update index_email        ← tambahan biaya
→ Update index_email_ispro  ← tambahan biaya lagi
```

---

### b. Apa Itu Duplicate Index?

- Duplicate index terjadi ketika dua index **memiliki kolom awal (leftmost prefix) yang sama**, sehingga keduanya bisa menjawab jenis query yang sama.
- Ini tidak selalu terlihat jelas — dua index bisa berbeda nama dan berbeda jumlah kolom, tapi tetap saling menduplikasi fungsinya.

**Contoh paling jelas (duplicate persis):**

```sql
-- Kedua index ini identik — jelas boros
CREATE INDEX idx_email_1 ON users (email);
CREATE INDEX idx_email_2 ON users (email);
```

---

### c. Duplicate Index yang Tidak Terlihat: Kasus Composite Index

Ini adalah kasus yang lebih berbahaya karena sering tidak disadari.

**Skenario umum:**

1. Awalnya dibuat index untuk kolom `email`:

```sql
CREATE INDEX idx_email ON users (email);
```

2. Suatu hari, ada kebutuhan baru untuk query berdasarkan `email` dan `is_pro` sekaligus. Lalu dibuat composite index:

```sql
CREATE INDEX idx_email_ispro ON users (email, is_pro);
```

3. Sekilas terlihat wajar, tapi sebenarnya `idx_email` kini **menjadi duplikat** dari `idx_email_ispro`.

**Visualisasi leftmost prefix:**

```
idx_email        → [ email ]
idx_email_ispro  → [ email | is_pro ]
                     ↑
                     Sama! (leftmost prefix identik)
```

---

### d. Cara Kerja B-Tree dan Aturan Leftmost Prefix

- B-tree index diakses dari **kiri ke kanan** berdasarkan urutan kolom yang didefinisikan.
- Sebuah composite index `(email, is_pro)` secara otomatis **bisa digunakan** untuk query yang hanya memfilter `email` saja, karena `email` adalah kolom pertama (leftmost).
- Ini berarti `idx_email_ispro` bisa menggantikan `idx_email` sepenuhnya, tapi tidak sebaliknya.

**Contoh query dan index yang digunakan:**

```sql
-- Query ini bisa menggunakan KEDUANYA:
SELECT * FROM users WHERE email = 'aaron@example.com';

-- Jika idx_email masih ada, query planner akan memilih yang lebih kecil/compact:
-- → menggunakan idx_email

-- Jika idx_email di-DROP, query planner beralih ke:
-- → menggunakan idx_email_ispro (tetap berfungsi!)

-- Query ini HANYA bisa menggunakan idx_email_ispro:
SELECT * FROM users WHERE email = 'aaron@example.com' AND is_pro = true;
```

---

### e. Solusi: Hapus Index yang Lebih Sempit

- Karena `idx_email_ispro` sudah bisa memenuhi semua kebutuhan `idx_email`, maka `idx_email` bisa **dihapus dengan aman**.
- `idx_email_ispro` adalah index yang **lebih kaya** — ia bisa melayani query `email` saja maupun query `email + is_pro`.
- Sebaliknya, `idx_email` tidak bisa melayani query `email + is_pro` secara efisien.

**Kesimpulan praktis:**

```sql
-- Hapus index yang lebih sempit:
DROP INDEX idx_email;

-- Pertahankan index yang lebih luas:
-- idx_email_ispro sudah cukup untuk kedua jenis query
```

> ⚠️ **Pengecualian:** Jika ada alasan teknis yang sangat spesifik (misalnya, ukuran index yang jauh lebih kecil sangat kritis untuk performa), kedua index mungkin masih dipertahankan. Tapi ini harus keputusan yang disengaja dan dipahami dengan jelas.

---

## 3. Hubungan Antar Konsep

Semua konsep dalam materi ini saling terhubung dalam satu alur pemahaman:

1. **Biaya index** → Setiap index menambah beban operasional, jadi harus diminimalkan.
2. **Cara kerja B-tree (leftmost prefix)** → Menentukan index mana yang bisa menjawab query tertentu. Ini adalah kunci untuk memahami mengapa composite index bisa "menelan" single-column index.
3. **Duplicate index tersembunyi** → Terjadi karena developer menambahkan composite index tanpa menyadari bahwa single-column index yang sudah ada menjadi redundan.
4. **Query planner** → Meskipun query planner cukup pintar untuk memilih index yang lebih kecil/efisien, ia tidak akan memperingatkan Anda bahwa ada index yang tidak diperlukan sama sekali.
5. **Solusi** → Hapus index yang lebih sempit (dan lebih mahal untuk dipelihara) karena index yang lebih luas sudah mencakup fungsinya.

---

## 4. Kesimpulan

Duplicate index adalah pemborosan sumber daya yang sering terjadi secara tidak sengaja, biasanya ketika composite index ditambahkan seiring perubahan kebutuhan bisnis tanpa menghapus single-column index yang sudah ada sebelumnya.

Kunci untuk menghindarinya adalah memahami **aturan leftmost prefix pada B-tree**: sebuah composite index `(A, B)` secara otomatis dapat menggantikan index `(A)` karena B-tree selalu dibaca dari kiri ke kanan. Jika dua index berbagi leftmost prefix yang sama, yang lebih sempit adalah kandidat untuk dihapus — kecuali ada alasan yang sangat spesifik untuk mempertahankannya.

> **Prinsip utama:** Index yang lebih luas `(email, is_pro)` selalu bisa melayani kebutuhan index yang lebih sempit `(email)`, tapi tidak sebaliknya.
