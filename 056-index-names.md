# 📝 Strategi Penamaan Index pada Database (Naming Indexes)

## 1. Ringkasan Singkat

Video ini membahas pentingnya memiliki strategi penamaan yang konsisten untuk _index_ dalam database relasional (khususnya PostgreSQL). Topik utama mencakup: ruang lingkup nama _index_ dalam _schema_, masalah konflik nama, pola penamaan yang direkomendasikan, dan pendekatan praktis yang digunakan ORM/framework modern.

---

## 2. Konsep Utama

### a. Ruang Lingkup Nama Index (Index Name Scope)

- Nama _index_ **tidak bersifat global** pada seluruh database.
- Namun, nama _index_ **bersifat global dalam satu schema**. Artinya, dua _index_ dengan nama yang sama **tidak bisa ada** dalam schema yang sama, meskipun berada di tabel yang berbeda.
- Contoh: jika kamu membuat _index_ bernama `email` pada tabel `users`, lalu mencoba membuat _index_ bernama `email` lagi pada tabel `addresses` (keduanya dalam schema `public`), maka operasi kedua akan **gagal**.

**Ilustrasi masalah:**

```sql
-- Schema: public
CREATE INDEX foo ON users(id);       -- ✅ Berhasil
CREATE INDEX foo ON addresses(id);   -- ❌ ERROR: index "foo" already exists
```

> Kesimpulan: nama _index_ harus **unik di dalam satu schema**.

---

### b. Pola Penamaan yang Direkomendasikan

Pola umum yang banyak digunakan adalah:

```
{nama_tabel}_{kolom_atau_kolom}_{tipe}
```

Komponen:

- **`nama_tabel`** — nama tabel tempat _index_ dibuat (prefix).
- **`kolom_atau_kolom`** — nama kolom atau kombinasi kolom yang diindeks.
- **`tipe`** — jenis _constraint_ atau _index_, misalnya:
  - `idx` → index biasa
  - `uniq` → unique index
  - `chk` → check constraint

**Contoh penamaan:**

| Tujuan                                       | Nama Index yang Baik |
| -------------------------------------------- | -------------------- |
| Index biasa pada kolom `id` di tabel `users` | `users_id_idx`       |
| Index pada kolom `email` di tabel `users`    | `users_email_idx`    |
| Unique index pada `email`                    | `users_email_uniq`   |

```sql
-- Index biasa pada kolom email di tabel users
CREATE INDEX users_email_idx ON users(email);

-- Unique index
CREATE UNIQUE INDEX users_email_uniq ON users(email);
```

---

### c. Mengapa Prefix Nama Tabel Penting?

- Menambahkan **nama tabel sebagai prefix** memberikan lapisan pemisahan yang jelas antar _index_.
- Memudahkan identifikasi: ketika melihat daftar _index_ dalam schema, kamu langsung tahu _index_ tersebut milik tabel apa.
- Mencegah konflik nama antar tabel secara alami, karena setiap nama menjadi spesifik per tabel.

---

### d. Peran ORM dan Framework

- Banyak framework modern yang menggunakan ORM (Object-Relational Mapping) — seperti Django, Laravel, Rails, dll. — **menghasilkan nama _index_ secara otomatis** saat melakukan migrasi database.
- Nama yang digenerate ORM biasanya sudah **dijamin unik**, sehingga tidak perlu khawatir soal konflik.
- Ini adalah pendekatan yang valid dan aman digunakan dalam proyek nyata.

**Contoh nama index yang digenerate framework (ilustrasi):**

```
index_users_on_email
users_pkey
fk_rails_abc123de
```

---

### e. Catatan Praktis untuk Video Tutorial

- Dalam konteks video pembelajaran, instruktur memilih untuk **tidak selalu mengetikkan nama _index_ yang panjang dan konvensional** demi efisiensi waktu.
- **Namun, di lingkungan production**, kamu **wajib** menggunakan nama _index_ yang bersih, deskriptif, dan mengikuti pola yang konsisten.
- Menggunakan nama pendek seperti `foo` hanya acceptable untuk eksperimen lokal atau latihan, **bukan untuk production**.

---

## 3. Hubungan Antar Konsep

Pemahaman tentang **ruang lingkup nama _index_** (poin a) adalah dasar dari segalanya — tanpa memahami bahwa nama _index_ bersifat unik per schema, kamu tidak akan mengerti mengapa strategi penamaan itu penting.

Dari situ, **pola penamaan konvensional** (poin b) hadir sebagai solusi praktis untuk menghindari konflik. **Penggunaan prefix nama tabel** (poin c) adalah bagian inti dari pola tersebut yang membuat nama menjadi lebih bermakna dan bebas tabrakan.

Di dunia nyata, **ORM dan framework** (poin d) sering menangani penamaan ini secara otomatis — namun pemahaman manual tetap penting ketika kamu menulis migrasi atau query DDL secara langsung.

Terakhir, **catatan praktis untuk tutorial** (poin e) mengingatkan bahwa ada perbedaan antara kode untuk belajar dan kode untuk production — kebiasaan baik harus dibawa ke lingkungan yang sesungguhnya.

---

## 4. Kesimpulan

- Nama _index_ **unik per schema**, bukan per database — ini sering menjadi sumber bug yang tidak terduga.
- Gunakan pola **`{tabel}_{kolom}_{tipe}`** sebagai konvensi penamaan yang jelas dan konsisten.
- Prefix nama tabel sangat direkomendasikan untuk memudahkan pengelolaan _index_.
- ORM/framework modern biasanya mengurus penamaan ini secara otomatis, dan itu sudah cukup baik.
- Dalam production, selalu gunakan nama _index_ yang deskriptif dan mengikuti konvensi tim atau framework yang digunakan.

---

> 💡 **Tips:** Konsistensi adalah kunci. Apapun konvensi yang kamu pilih bersama tim, pastikan semua orang mengikutinya agar _schema_ database tetap mudah dibaca dan dikelola.
