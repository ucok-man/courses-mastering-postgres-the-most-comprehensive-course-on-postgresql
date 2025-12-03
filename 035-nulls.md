# 📝 Nulls dan Constraint NOT NULL di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang constraint NOT NULL di PostgreSQL, dengan penekanan pada karakteristik unik dari nilai NULL dalam database dan best practices dalam penggunaannya. Materi ini merupakan bagian akhir dari modul tentang tipe data dan constraint dasar.

## 2. Konsep Utama

### a. Karakteristik Khusus NULL

NULL memiliki sifat yang "aneh" (weird) dibandingkan nilai lainnya:

- **NULL bukan nilai kosong biasa**: NULL ≠ 0, NULL ≠ false, NULL ≠ empty string, NULL ≠ empty
- **NULL adalah "unknown value"**: Nilai yang tidak diketahui atau tidak dapat diketahui

**Analogi perbandingan dengan NULL:**

Ketika database diminta membandingkan sesuatu dengan NULL, seperti:

```sql
SELECT 1 = NULL;  -- Hasilnya bukan TRUE atau FALSE, tapi NULL (unknown)
```

Ini seperti bertanya: "Apakah angka 1 sama dengan benda rahasia yang saya sembunyikan di tangan?"

Database hanya bisa menjawab: "Saya tidak tahu apa yang Anda sembunyikan, jadi saya tidak bisa membandingkannya."

**Cara yang benar untuk memeriksa NULL:**

```sql
-- ❌ SALAH: Tidak akan bekerja seperti yang diharapkan
SELECT * FROM products WHERE name = NULL;

-- ✅ BENAR: Menggunakan IS NULL
SELECT * FROM products WHERE name IS NULL;
```

### b. Three-Valued Boolean Logic

Ketika NULL terlibat dalam perbandingan, hasilnya bukan hanya TRUE atau FALSE, tetapi ada kemungkinan ketiga: **NULL (unknown)**.

```sql
SELECT 'Aaron' = NULL;  -- Hasil: NULL (bukan TRUE atau FALSE)
SELECT NULL = NULL;     -- Hasil: NULL (bahkan NULL tidak "equal" dengan NULL)
```

### c. Constraint NOT NULL

**Sintaks dasar:**

```sql
-- ❌ Cara yang tidak direkomendasikan (menggunakan CHECK constraint)
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT CHECK (name IS NOT NULL)  -- Tidak efisien
);

-- ✅ Cara yang direkomendasikan
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,  -- Lebih efisien dan jelas
    price NUMERIC NOT NULL
);
```

**Kombinasi dengan CHECK constraint:**

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC NOT NULL CHECK (price > 0)  -- Menggabungkan NOT NULL dan domain logic
);
```

Pada contoh di atas:

- `NOT NULL` memastikan harus ada nilai
- `CHECK (price > 0)` memastikan nilai harus lebih besar dari 0 (karena untuk bisnis tertentu, 0 sama saja dengan NULL)

### d. Primary Key dan NOT NULL

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
```

**Poin penting:**

- PRIMARY KEY secara otomatis menambahkan constraint NOT NULL
- Tidak perlu menambahkan NOT NULL secara eksplisit pada kolom primary key

### e. Perilaku Default (Nullable)

**Secara default, semua kolom adalah NULLABLE** kecuali dinyatakan sebaliknya:

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT,        -- Nullable (default)
    description TEXT  -- Nullable (default)
);

-- Ini valid:
INSERT INTO products (name) VALUES (NULL);  -- ✅ Berhasil
```

## 3. Hubungan Antar Konsep

### Alur Pemahaman:

1. **NULL sebagai fondasi**: Memahami bahwa NULL adalah "unknown value" yang berperilaku berbeda dari nilai normal
2. **Implikasi three-valued logic**: Karena NULL adalah unknown, operasi perbandingan menghasilkan NULL (bukan TRUE/FALSE)
3. **Kebutuhan akan NOT NULL**: Untuk menghindari kompleksitas NULL, gunakan constraint NOT NULL
4. **Manfaat NOT NULL**: Dengan membatasi NULL, operasi seperti:

   - Indexing
   - Comparing
   - Grouping
   - Sorting
   - Querying

   Menjadi lebih sederhana dan efisien

### Best Practice Decision Tree:

```
Apakah kolom ini secara logika HARUS memiliki nilai?
├─ YA → Gunakan NOT NULL
└─ TIDAK → Biarkan nullable (default)
    └─ JANGAN membuat "fake null" (seperti menggunakan string kosong untuk merepresentasikan NULL)
```

## 4. Kesimpulan

- **NULL adalah "unknown value"**, bukan nilai kosong biasa (bukan 0, false, atau empty string)
- Perbandingan dengan NULL menghasilkan **three-valued logic** (TRUE, FALSE, atau NULL)
- **Best practice**: Jadikan NOT NULL sebagai default untuk kolom Anda, kecuali ada alasan kuat untuk membiarkannya nullable
- **Sintaks yang benar**: `column_name TYPE NOT NULL` (bukan menggunakan CHECK constraint)
- **Manfaat NOT NULL**: Enforcement nilai wajib ada + query operations yang lebih efisien
- Constraint NOT NULL bisa dikombinasikan dengan CHECK constraint untuk domain logic tambahan
- PRIMARY KEY secara otomatis adalah NOT NULL

**Prinsip emas**: "Make every column NOT NULL unless you have a very good reason to make it nullable."
