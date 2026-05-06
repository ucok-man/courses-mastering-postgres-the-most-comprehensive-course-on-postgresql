# 📝 Ordering NULLs dalam Index (PostgreSQL)

## 1. Ringkasan Singkat

Video ini membahas bagaimana PostgreSQL menangani nilai `NULL` saat melakukan pengurutan (`ORDER BY`), baik dalam query biasa maupun dalam konstruksi index. Secara default, `NULL` diperlakukan sebagai nilai terbesar, namun perilaku ini bisa diubah menggunakan klausa `NULLS FIRST` atau `NULLS LAST` — baik di level query maupun di level definisi index itu sendiri.

---

## 2. Konsep Utama

### a. Perilaku Default NULL dalam Pengurutan

- Secara default di PostgreSQL, `NULL` dianggap **lebih besar dari nilai apapun**.
- Konsekuensinya:
  - Pada `ORDER BY kolom ASC` → NULL muncul di **paling bawah** (karena dianggap terbesar).
  - Pada `ORDER BY kolom DESC` → NULL muncul di **paling atas** (karena dianggap terbesar dan DESC membalik urutan).

**Contoh:**

```sql
-- NULL muncul di bawah (ascending = terkecil dulu)
SELECT * FROM users ORDER BY birthday ASC;

-- NULL muncul di atas (descending = terbesar dulu)
SELECT * FROM users ORDER BY birthday DESC;
```

> 💡 **Tips:** Ini sering membingungkan — NULL ada di bawah saat ASC dan di atas saat DESC.

---

### b. Klausa NULLS FIRST dan NULLS LAST

PostgreSQL menyediakan klausa opsional untuk mengontrol posisi NULL secara eksplisit, terlepas dari arah pengurutan:

| Kombinasi                       | Posisi NULL  | Catatan                                  |
| ------------------------------- | ------------ | ---------------------------------------- |
| `ORDER BY col ASC`              | Paling bawah | Default                                  |
| `ORDER BY col DESC`             | Paling atas  | Default                                  |
| `ORDER BY col DESC NULLS LAST`  | Paling bawah | **Override** dari default DESC           |
| `ORDER BY col ASC NULLS FIRST`  | Paling atas  | **Override** dari default ASC            |
| `ORDER BY col DESC NULLS FIRST` | Paling atas  | Sama dengan default DESC (tidak berguna) |
| `ORDER BY col ASC NULLS LAST`   | Paling bawah | Sama dengan default ASC (tidak berguna)  |

**Contoh penggunaan yang bermakna:**

```sql
-- NULL dipaksa ke bawah meskipun DESC
SELECT * FROM users ORDER BY birthday DESC NULLS LAST;

-- NULL dipaksa ke atas meskipun ASC
SELECT * FROM users ORDER BY birthday ASC NULLS FIRST;
```

---

### c. Membuat Index yang Mencerminkan Urutan NULL

Jika query sering menggunakan `NULLS FIRST` atau `NULLS LAST`, maka sebaiknya dibuat index yang **mencerminkan urutan yang sama** agar index bisa digunakan secara efisien (bukan fallback ke sequential scan).

**Contoh membuat index dengan NULLS FIRST:**

```sql
CREATE INDEX birthday_nulls_first ON users (birthday ASC NULLS FIRST);
```

**Cara memverifikasi index digunakan:**

```sql
EXPLAIN SELECT * FROM users ORDER BY birthday ASC NULLS FIRST LIMIT 10;
-- Hasil: Index Scan on birthday_nulls_first
```

---

### d. Aturan Index Scan: Forward vs Backward

Index yang dibuat dengan satu konfigurasi tertentu bisa digunakan **maju (forward)** maupun **mundur (backward)** — namun dengan syarat:

- Jika query **membalik arah pengurutan (ASC ↔ DESC) sekaligus juga membalik posisi NULL** → PostgreSQL menggunakan **Backward Index Scan** ✅
- Jika query hanya membalik **salah satu** (misalnya hanya arah, tapi tidak posisi NULL, atau sebaliknya) → Index **tidak cocok**, PostgreSQL jatuh ke **Sequential Scan** ❌

**Contoh:**

```sql
-- Index: birthday ASC NULLS FIRST

-- ✅ Menggunakan Backward Index Scan (keduanya dibalik)
ORDER BY birthday DESC NULLS LAST

-- ❌ Sequential Scan (hanya arah yang dibalik, NULL tidak dibalik)
ORDER BY birthday DESC NULLS FIRST

-- ❌ Sequential Scan (hanya NULL yang dibalik, arah tidak berubah)
ORDER BY birthday ASC NULLS LAST
```

> 💡 **Aturan kunci:** Agar backward index scan bisa bekerja, **kedua** hal harus dibalik secara konsisten: arah (ASC/DESC) **dan** posisi NULL (FIRST/LAST).

---

## 3. Hubungan Antar Konsep

Alur pemahaman dari video ini membentuk hierarki berikut:

```
Perilaku Default NULL
        │
        ▼
Klausa NULLS FIRST / NULLS LAST di Query
        │
        ▼
Membuat Index yang Cocok dengan Urutan Query
        │
        ▼
Aturan Forward/Backward Index Scan
```

- Memahami **perilaku default** NULL adalah fondasi — tanpanya, urutan hasil query bisa mengejutkan.
- Klausa `NULLS FIRST/LAST` memberi **kontrol eksplisit** atas posisi NULL.
- Agar query tetap cepat (menggunakan index, bukan seq scan), index harus **dibuat dengan urutan yang identik** dengan yang digunakan query.
- Aturan **forward vs backward scan** menjelaskan mengapa hanya membalik salah satu aspek saja tidak cukup — keduanya harus konsisten.

---

## 4. Kesimpulan

- `NULL` di PostgreSQL secara default dianggap **nilai terbesar** dalam pengurutan.
- Gunakan `NULLS FIRST` atau `NULLS LAST` di query untuk mengontrol posisi NULL secara eksplisit.
- Jika pola pengurutan dengan `NULLS FIRST/LAST` digunakan secara rutin, buat index yang **mencerminkan urutan yang sama** untuk performa optimal.
- Saat menggunakan index secara backward (membalik arah), pastikan **arah dan posisi NULL keduanya dibalik** agar PostgreSQL tetap bisa memanfaatkan index tersebut.
- Jika hanya salah satu yang dibalik, PostgreSQL akan melakukan **sequential scan** yang jauh lebih lambat.
