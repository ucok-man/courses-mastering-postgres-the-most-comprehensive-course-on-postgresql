# 📝 Full Text Search di PostgreSQL: TS Vector dan TS Query

## 1. Ringkasan Singkat

Video ini membahas cara menggunakan fitur **full text search** bawaan PostgreSQL tanpa perlu tools eksternal seperti Elasticsearch. Fokus pembahasan adalah dua tipe data khusus: **`tsvector`** dan **`tsquery`**, serta cara mengimplementasikannya dengan **generated columns** agar data pencarian selalu sinkron dengan data asli.

---

## 2. Konsep Utama

### a. Full Text Search di PostgreSQL

- PostgreSQL menyediakan fitur full text search yang bisa mencakup **80-90% kebutuhan** pencarian teks.
- Keuntungannya: **stack aplikasi tetap sederhana**, tidak perlu menambah service terpisah seperti Elasticsearch atau MeiliSearch.
- Keterbatasan: Untuk fitur advanced seperti **faceted search** atau **highlighting hasil**, mungkin perlu solusi eksternal.
- Beberapa hosting provider (contoh: Zeta) menawarkan replikasi otomatis dari Postgres ke Elasticsearch.

---

### b. Tipe Data `tsvector`

**Definisi:**
`tsvector` adalah tipe data yang menyimpan **sorted list of distinct lexemes** (daftar lexeme unik yang sudah diurutkan).

**Apa itu lexeme?**

- Lexeme adalah **unit dasar bahasa** yang sudah dinormalisasi.
- Contoh: kata "lazy" dan "laziness" akan di-**stem** menjadi lexeme yang sama: `lazi`.
- Ini memberikan **fuzzy matching** otomatis tanpa konfigurasi tambahan.

**Cara membuat `tsvector`:**

```sql
SELECT to_tsvector('the quick brown fox');
-- Output: 'brown':3 'fox':4 'quick':2
```

Perhatikan:

- Kata-kata diurutkan alfabetis
- Angka di sebelah kata menunjukkan **posisi** dalam teks asli
- Kata umum seperti "the" dihilangkan (stop words)

**Contoh dengan kata yang sama muncul dua kali:**

```sql
SELECT to_tsvector('the quick brown fox jumps over the lazy fox');
-- 'fox' akan muncul sekali dengan posisi 4 dan 9: 'fox':4,9
```

**Contoh lexeme normalization:**

```sql
SELECT to_tsvector('lazy laziness');
-- Output: 'lazi':1,2
-- Kedua kata dinormalisasi jadi lexeme yang sama
```

---

### c. Tipe Data `tsquery`

**Definisi:**
`tsquery` adalah tipe data untuk **query pencarian** yang juga akan dinormalisasi menjadi lexeme.

**Cara membuat `tsquery`:**

```sql
SELECT to_tsquery('lazy');
-- Output: 'lazi'

SELECT to_tsquery('laziness');
-- Output: 'lazi' (sama dengan 'lazy')
```

---

### d. Operator Pencarian: `@@`

Operator `@@` digunakan untuk **mencocokkan** `tsquery` dengan `tsvector`.

**Sintaks:**

```sql
tsvector @@ tsquery
```

**Contoh penggunaan:**

```sql
-- Cocok: kata "lazy" ada di teks
SELECT to_tsvector('the quick brown fox jumps over the lazy dog')
       @@ to_tsquery('lazy');
-- Output: true

-- Cocok: meskipun query "laziness" tapi teks ada "lazy" (lexeme sama)
SELECT to_tsvector('the quick brown fox jumps over the lazy dog')
       @@ to_tsquery('laziness');
-- Output: true

-- Tidak cocok: kata "red" tidak ada
SELECT to_tsvector('the quick brown fox jumps over the lazy dog')
       @@ to_tsquery('red');
-- Output: false
```

**Catatan penting:**

- Urutan `tsvector` dan `tsquery` bisa dibalik, hasil tetap sama
- Ada banyak operator lain (AND, OR, NOT, weighting, ranking) yang akan dibahas di modul terpisah

---

### e. Konfigurasi Bahasa

Fungsi `to_tsvector()` bisa menerima parameter **bahasa** untuk menentukan aturan stemming dan stop words.

**Sintaks:**

```sql
SELECT to_tsvector('english', 'the quick brown fox');
SELECT to_tsvector('french', 'bonjour'); -- contoh bahasa Prancis
```

**Kenapa penting?**

- Setiap bahasa punya aturan stemming berbeda
- Default-nya menggunakan `default_text_search_config` (biasanya English)
- Untuk generated columns, **wajib specify bahasa** agar fungsi bersifat **immutable**

---

### f. Implementasi dengan Generated Columns

**Masalah:** Jika `tsvector` disimpan sebagai kolom terpisah, bisa **tidak sinkron** dengan kolom teks asli.

**Solusi:** Gunakan **generated column** yang otomatis menghitung ulang `tsvector` setiap kali data berubah.

**Contoh tabel:**

```sql
CREATE TABLE ts_example (
  id SERIAL PRIMARY KEY,
  content TEXT,
  search_vector_en TSVECTOR
    GENERATED ALWAYS AS (to_tsvector('english', content)) STORED
);
```

**Penjelasan:**

- `GENERATED ALWAYS AS`: kolom ini akan dihitung otomatis
- `to_tsvector('english', content)`: konversi kolom `content` jadi `tsvector`
- `STORED`: hasil disimpan di disk (PostgreSQL tidak support `VIRTUAL`)
- **Wajib specify 'english'**: tanpa ini, fungsi dianggap **non-immutable** dan akan error

**Kenapa harus immutable?**
Jika tidak specify bahasa, hasil `to_tsvector()` bisa berubah jika admin mengubah `default_text_search_config`. Generated columns harus **deterministic** (input sama = output sama).

---

### g. Query dengan Generated Column

**Insert data:**

```sql
INSERT INTO ts_example (content)
VALUES ('the quick brown fox jumps over the lazy dog');

INSERT INTO ts_example (content)
VALUES ('the quick brown fox jumps over the cat');
```

**Query pencarian:**

```sql
-- Cari semua baris yang mengandung kata "lazy"
SELECT * FROM ts_example
WHERE search_vector_en @@ to_tsquery('lazy');
-- Output: hanya baris pertama

-- Cari semua baris yang mengandung kata "cat"
SELECT * FROM ts_example
WHERE search_vector_en @@ to_tsquery('cat');
-- Output: hanya baris kedua
```

**Keuntungan:**

- Hanya perlu **insert ke kolom `content`** saja
- Kolom `search_vector_en` otomatis ter-update
- Tidak perlu trigger atau manual update
- Tidak mungkin out-of-sync

---

## 3. Hubungan Antar Konsep

```
┌─────────────┐
│ Text Asli   │ (disimpan di kolom TEXT)
└──────┬──────┘
       │
       │ to_tsvector('english', content)
       ▼
┌─────────────┐
│  tsvector   │ (generated column, normalized & indexed)
└──────┬──────┘
       │
       │ operator @@
       ▼
┌─────────────┐
│  tsquery    │ (query yang juga di-normalize)
└─────────────┘
       │
       ▼
  [true/false]
```

**Alur kerja:**

1. User menyimpan teks asli di kolom `content`
2. PostgreSQL otomatis mengkonversi ke `tsvector` via generated column
3. Saat search, query user dikonversi ke `tsquery`
4. Operator `@@` mencocokkan `tsvector` dengan `tsquery`
5. Return `true` jika match, `false` jika tidak

**Keuntungan pendekatan ini:**

- **Single source of truth**: hanya kolom `content` yang di-update manual
- **Always in sync**: `tsvector` pasti sesuai dengan `content`
- **Performance**: `tsvector` bisa di-index untuk pencarian cepat
- **Fuzzy matching gratis**: stemming otomatis (lazy = laziness)

---

## 4. Kesimpulan

PostgreSQL menyediakan fitur full text search yang powerful lewat dua tipe data khusus:

- **`tsvector`**: representasi teks yang sudah dinormalisasi dan dioptimalkan untuk pencarian
- **`tsquery`**: query pencarian yang juga dinormalisasi

Dengan kombinasi **generated columns**, kita bisa memastikan data pencarian selalu sinkron tanpa perlu trigger atau maintenance manual. Fitur ini mencakup 80-90% kebutuhan full text search tanpa perlu menambah kompleksitas infrastruktur.

**Next steps:**

- Pelajari operator advanced (AND, OR, NOT)
- Ranking dan weighting hasil pencarian
- Stop words dan konfigurasi bahasa
- Indexing strategy untuk performa optimal
