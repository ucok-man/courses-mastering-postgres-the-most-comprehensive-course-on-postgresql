# 📝 Hash Indexes di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas **hash index**, salah satu jenis index di PostgreSQL selain B-tree index. Hash index bekerja dengan cara mengubah nilai kolom melalui _hashing function_ sebelum disimpan di struktur index. Karena sifat inilah, hash index hanya cocok untuk satu jenis operasi saja: **strict equality lookup** (pencarian dengan operator `=`). Meski terbatas, hash index menawarkan performa yang sangat baik untuk kasus penggunaan tersebut, dan ukuran index-nya pun relatif kecil dan efisien.

---

## 2. Konsep Utama

### a. Apa Itu Hash Index?

- Hash index adalah tipe index alternatif dari B-tree index di PostgreSQL.
- Nilai kolom yang diindex **tidak disimpan langsung**—melainkan dijalankan melalui sebuah _hashing function_ terlebih dahulu.
- Hasil dari _hashing function_ ini yang disimpan di dalam struktur index.
- Karena nilai asli tidak tersimpan (hanya hasil hash-nya), **makna semantik nilai tersebut hilang** setelah proses hashing.

**Ilustrasi alur:**

```
Nilai asli: "aaron.francis@example.com"
          ↓ hashing function
Hash:      0xA3F7C2B1...  ← ini yang disimpan di index
```

---

### b. Keterbatasan Hash Index

Karena nilai yang tersimpan adalah _hash_ (bukan nilai aslinya), hash index **hanya bisa digunakan untuk strict equality lookup** (`=`). Hash index **tidak bisa** digunakan untuk:

| Operasi                                  | Bisa Digunakan? |
| ---------------------------------------- | --------------- |
| Strict equality (`=`)                    | ✅ Ya           |
| Range search (`<`, `>`, `BETWEEN`)       | ❌ Tidak        |
| Wildcard / partial match (`LIKE 'abc%'`) | ❌ Tidak        |
| Ordering / sorting (`ORDER BY`)          | ❌ Tidak        |

**Alasan:** Setelah nilai di-hash, tidak ada cara untuk membandingkan apakah satu hash "lebih besar" atau "lebih kecil" dari hash lainnya secara bermakna. Dua hash yang berbeda hanya bisa dinyatakan _sama_ atau _berbeda_.

---

### c. Keunggulan Hash Index

**1. Lebih cepat untuk strict equality lookup**

Hash index menggunakan struktur data yang berbeda dari B-tree dan **dioptimalkan khusus** untuk menyimpan dan mencari hash. Ini membuatnya lebih cepat dari B-tree untuk query `=`.

Analogi konseptual:

```sql
-- Hash index mirip seperti B-tree index di atas MD5, tapi lebih optimal:
CREATE INDEX ON users (MD5(email)); -- analogi saja, bukan kode nyata

-- Hash index yang sesungguhnya:
CREATE INDEX email_hash ON users USING hash (email);
```

**2. Ukuran index yang konstan dan kecil**

Nilai apapun yang di-hash—baik teks pendek maupun teks panjang—akan menghasilkan output dengan **ukuran yang sama (constant size)**. Ini berarti:

- Kolom berisi URL yang sangat panjang (ribuan karakter) akan menghasilkan hash berukuran sama dengan kolom berisi teks pendek.
- Struktur index tetap **kecil dan ringkas**, tidak peduli seberapa besar nilai yang diindex.

```
"hi"                              → hash (32 byte)
"https://example.com/page?utm_source=...sangat_panjang..." → hash (32 byte)
```

---

### d. Cara Membuat Hash Index di PostgreSQL

Gunakan sintaks `USING hash` saat membuat index:

```sql
-- Membuat B-tree index (default)
CREATE INDEX email_btree ON users USING btree (email);

-- Membuat Hash index
CREATE INDEX email_hash ON users USING hash (email);
```

Kata kunci `USING` adalah bagian penting yang membedakan jenis index yang dibuat.

---

### e. Bagaimana PostgreSQL Memilih Index

Ketika ada dua index (B-tree dan hash) pada kolom yang sama, PostgreSQL akan **memilih index yang paling sesuai** secara otomatis berdasarkan jenis query:

```sql
-- Query ini akan menggunakan hash index (strict equality)
SELECT * FROM users WHERE email = 'aaron.francis@example.com';

-- Query ini akan menggunakan B-tree index (range)
SELECT * FROM users WHERE email < 'aaron.francis@example.com';

-- Query ini tidak menggunakan index hash MAUPUN B-tree (tergantung estimasi planner)
-- bisa jadi sequential scan
SELECT * FROM users WHERE email LIKE 'aaron.francis@%';
```

PostgreSQL cukup cerdas untuk **tidak pernah menggunakan hash index pada operasi selain `=`** karena secara teknis hal itu tidak valid/legal.

---

### f. Kapan Sebaiknya Menggunakan Hash Index?

Hash index cocok digunakan ketika:

1. **Query yang sering dijalankan hanya menggunakan operator `=`** (strict equality).
2. **Kolom yang diindex berukuran besar**, misalnya:
   - URL (bisa ribuan karakter dengan UTM parameter, dsb.)
   - Alamat email
   - API token / secret key
   - Blob teks panjang lainnya

**Contoh use case:**

```sql
-- Lookup cepat berdasarkan URL (bisa sangat panjang)
CREATE INDEX url_hash ON pages USING hash (url);

-- Lookup berdasarkan API token
CREATE INDEX token_hash ON sessions USING hash (api_token);
```

---

### g. Peringatan: Versi PostgreSQL

> ⚠️ **Penting!** Sebelum **PostgreSQL versi 10**, hash index **sangat berbahaya** dan sebaiknya tidak digunakan sama sekali.

Masalah pada versi lama (< PostgreSQL 10):

- Hash index **tidak ditulis ke Write-Ahead Log (WAL)**, sehingga tidak ikut direplikasi ke replica server.
- Bisa menyebabkan **crash** dan ketidakkonsistenan data.
- Setelah PostgreSQL 10, semua masalah ini telah diperbaiki.

**Kesimpulan:** Jika menggunakan PostgreSQL 10 ke atas (saat ini sudah PostgreSQL 17), hash index **aman digunakan**.

---

## 3. Hubungan Antar Konsep

Pemahaman hash index akan lebih utuh jika dikaitkan dengan B-tree index sebagai perbandingan:

```
                    ┌─────────────────────────────────────┐
                    │           INDEX DI POSTGRESQL        │
                    └────────────────┬────────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              ▼                                             ▼
     ┌─────────────────┐                         ┌──────────────────┐
     │   B-tree Index  │                         │   Hash Index     │
     └────────┬────────┘                         └────────┬─────────┘
              │                                           │
     Menyimpan nilai asli                      Menyimpan hasil hash
     (terurut / sorted)                        (ukuran konstan)
              │                                           │
     Mendukung:                                 Hanya mendukung:
     - equality (=)                             - equality (=)
     - range (<, >, BETWEEN)
     - LIKE 'abc%'
     - ORDER BY
```

- Hash index adalah **spesialisasi**: ia mengorbankan fleksibilitas demi kecepatan dan efisiensi memori pada kasus yang sangat spesifik.
- B-tree index adalah pilihan **default dan general-purpose**.
- PostgreSQL query planner akan **secara otomatis memilih** index mana yang paling optimal, sehingga developer tidak perlu memilih secara manual dalam query.

---

## 4. Kesimpulan

Hash index adalah tipe index di PostgreSQL yang **sangat efisien untuk strict equality lookup** (`=`), namun **tidak bisa digunakan** untuk range query, wildcard, atau sorting. Cara kerjanya: nilai kolom di-hash terlebih dahulu sehingga ukuran index tetap kecil dan konstan, cocok untuk kolom berisi data panjang seperti URL atau API token.

**Poin-poin kunci:**

- Hanya untuk operator `=` — tidak untuk `<`, `>`, `LIKE`, `ORDER BY`, dsb.
- Lebih cepat dari B-tree untuk strict equality.
- Ukuran index lebih kecil karena hash selalu berukuran konstan.
- **Aman digunakan mulai PostgreSQL 10** ke atas.
- PostgreSQL memilih index yang tepat secara otomatis.
- Pertimbangkan untuk kolom seperti URL panjang, email, atau API token yang sering di-query dengan `=`.
- Selalu benchmark dan bandingkan antara hash dan B-tree untuk kebutuhan spesifik Anda.
