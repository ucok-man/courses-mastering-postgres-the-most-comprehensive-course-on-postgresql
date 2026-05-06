# 📝 Urutan Kolom & Arah Pengurutan pada Index di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang bagaimana **urutan dan arah pengurutan (ascending/descending)** saat membuat index — terutama _composite index_ — memengaruhi performa query di PostgreSQL. Poin utamanya adalah: untuk index dengan satu kolom, PostgreSQL cukup fleksibel karena bisa membaca index maju maupun mundur. Namun, untuk _composite index_, urutan arah pengurutan tiap kolom harus **sesuai persis** dengan pola `ORDER BY` yang digunakan pada query, atau PostgreSQL terpaksa melakukan _incremental sort_ yang lebih lambat.

---

## 2. Konsep Utama

### a. Membuat Index dengan Urutan Tertentu

Saat membuat index, kita bisa menentukan apakah kolom diurutkan secara **ascending (ASC)** atau **descending (DESC)**. Secara default, PostgreSQL menggunakan ascending.

**Contoh — membuat index ascending (default):**

```sql
CREATE INDEX created_at ON users (created_at);
-- Sama dengan:
CREATE INDEX created_at ON users (created_at ASC);
```

**Contoh — membuat index descending:**

```sql
CREATE INDEX created_at_desc ON users (created_at DESC);
```

Tujuannya adalah agar urutan index cocok dengan pola `ORDER BY` yang sering digunakan dalam query, sehingga PostgreSQL tidak perlu melakukan sorting tambahan.

---

### b. Index Scan Maju dan Mundur pada Single-Column Index

PostgreSQL memiliki kemampuan membaca index dari **depan ke belakang (forward scan)** maupun **belakang ke depan (backward scan)**. Ini berarti untuk index satu kolom, urutan pembuatannya tidak terlalu kritis — PostgreSQL bisa menyesuaikan arahnya secara otomatis.

**Contoh:**

```sql
-- Index dibuat ascending
CREATE INDEX created_at ON users (created_at);

-- Query ascending → index scan normal (forward)
EXPLAIN SELECT * FROM users ORDER BY created_at LIMIT 10;
-- Output: Index Scan using created_at on users

-- Query descending → index scan mundur (backward)
EXPLAIN SELECT * FROM users ORDER BY created_at DESC LIMIT 10;
-- Output: Index Scan Backward using created_at on users
```

> **Kesimpulan untuk single-column index:** Tidak perlu khawatir soal urutan; PostgreSQL akan membaca index ke arah mana pun yang dibutuhkan.

---

### c. Composite Index dan Masalah Arah yang Tidak Seragam

Pada **composite index** (index dengan lebih dari satu kolom), PostgreSQL hanya bisa membaca keseluruhan index ke **satu arah** — maju semua atau mundur semua. Index tidak bisa membaca sebagian kolom maju dan sebagian lagi mundur.

**Ilustrasi struktur composite index:**

```
Index: (birthday ASC, created_at ASC)
Dibaca maju:   birthday ↑, created_at ↑  ✅
Dibaca mundur: birthday ↓, created_at ↓  ✅
Campuran:      birthday ↑, created_at ↓  ❌ → Incremental Sort!
```

**Contoh skenario bermasalah:**

```sql
-- Index dibuat: birthday ASC, created_at ASC (keduanya default)
CREATE INDEX birthday_created_at ON users (birthday, created_at);

-- Query 1: keduanya ASC → OK ✅
EXPLAIN SELECT * FROM users ORDER BY birthday ASC, created_at ASC LIMIT 10;
-- Output: Index Scan using birthday_created_at on users

-- Query 2: keduanya DESC → OK ✅ (backward scan)
EXPLAIN SELECT * FROM users ORDER BY birthday DESC, created_at DESC LIMIT 10;
-- Output: Index Scan Backward using birthday_created_at on users

-- Query 3: birthday DESC, created_at ASC → BERMASALAH ❌
EXPLAIN SELECT * FROM users ORDER BY birthday DESC, created_at ASC LIMIT 10;
-- Output: Incremental Sort  ← ini tanda bahwa index tidak bisa digunakan sepenuhnya!

-- Query 4: birthday ASC, created_at DESC → BERMASALAH ❌
EXPLAIN SELECT * FROM users ORDER BY birthday ASC, created_at DESC LIMIT 10;
-- Output: Incremental Sort
```

**Mengapa _incremental sort_ itu buruk?**
PostgreSQL mengambil _block_ data dari index, lalu melakukan sorting lagi di memori. Ini menambah overhead CPU dan memori, terutama pada dataset besar.

---

### d. Solusi: Membuat Composite Index dengan Arah yang Tepat

Jika query yang sering digunakan menggunakan arah pengurutan yang berbeda antar kolom (misalnya `birthday ASC, created_at DESC`), kita harus **mendefinisikan index sesuai dengan pola tersebut**.

**Langkah-langkah:**

```sql
-- Hapus index yang tidak cocok
DROP INDEX birthday_created_at;

-- Buat ulang dengan arah yang sesuai pola query
CREATE INDEX birthday_created_at ON users (birthday ASC, created_at DESC);
```

**Verifikasi dengan EXPLAIN:**

```sql
-- Query dengan pola yang sama dengan index → OK ✅
EXPLAIN SELECT * FROM users ORDER BY birthday ASC, created_at DESC LIMIT 10;
-- Output: Index Scan using birthday_created_at on users  ← incremental sort hilang!

-- Keduanya dibalik → masih OK ✅ (backward scan)
EXPLAIN SELECT * FROM users ORDER BY birthday DESC, created_at ASC LIMIT 10;
-- Output: Index Scan Backward using birthday_created_at on users

-- Hanya satu yang dibalik → BERMASALAH ❌
EXPLAIN SELECT * FROM users ORDER BY birthday DESC, created_at DESC LIMIT 10;
-- Output: Incremental Sort
```

> **Aturan kunci:** Index composite bisa dibaca maju _atau_ mundur secara **keseluruhan**. Kamu tidak bisa "membalik" hanya sebagian kolom saja.

---

### e. Cara Melihat Index yang Ada

Untuk memeriksa index yang sudah dibuat pada suatu tabel, gunakan:

```sql
SELECT * FROM pg_indexes WHERE tablename = 'users';
```

Ini berguna untuk memverifikasi nama, kolom, dan urutan index yang aktif.

---

## 3. Hubungan Antar Konsep

Alur pemahaman dalam video ini membentuk logika yang bertahap:

1. **Index dasar** → Kita bisa menentukan urutan ASC/DESC saat membuat index, biasanya agar cocok dengan pola query.
2. **Single-column index** → Fleksibel; PostgreSQL otomatis melakukan _forward_ atau _backward scan_ sesuai kebutuhan query. Tidak ada masalah di sini.
3. **Composite index** → Lebih ketat. Karena index adalah satu struktur tunggal, PostgreSQL hanya bisa membacanya ke satu arah (maju atau mundur seluruhnya).
4. **Masalah arah campuran** → Jika query menggunakan arah berbeda untuk tiap kolom (misalnya ASC + DESC), index tidak bisa digunakan optimal → muncul _incremental sort_.
5. **Solusi** → Buat index dengan urutan arah yang **persis sama** dengan pola `ORDER BY` yang paling sering digunakan. Dengan begitu, index bisa digunakan secara penuh, baik secara forward maupun backward scan.

---

## 4. Kesimpulan

| Tipe Index                    | Fleksibilitas Arah | Catatan                                                         |
| ----------------------------- | ------------------ | --------------------------------------------------------------- |
| **Single-column**             | Tinggi             | Bisa forward & backward scan otomatis                           |
| **Composite (arah seragam)**  | Tinggi             | Bisa forward & backward scan seluruhnya                         |
| **Composite (arah campuran)** | Rendah             | Harus dibuat sesuai pola query; jika tidak → _incremental sort_ |

**Poin penting yang perlu diingat:**

- Untuk _single-column index_, urutan ASC/DESC saat pembuatan tidak kritis — PostgreSQL cukup cerdas untuk menyesuaikan.
- Untuk _composite index_, pastikan arah pengurutan tiap kolom dalam definisi index **cocok** dengan pola `ORDER BY` yang paling sering digunakan dalam query.
- Kamu boleh membalik **semua** kolom sekaligus (karena itu hanya backward scan), tetapi **tidak boleh** membalik sebagian kolom saja.
- Gunakan `EXPLAIN` secara rutin untuk memverifikasi apakah query menggunakan _index scan_ atau _incremental sort_. Kehadiran _incremental sort_ adalah sinyal bahwa definisi index perlu ditinjau ulang.
