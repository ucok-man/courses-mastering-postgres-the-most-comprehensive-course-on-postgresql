# 📝 Tipe Data Numeric (DECIMAL) di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tipe data `NUMERIC` (alias `DECIMAL`) di PostgreSQL, yang merupakan tipe data untuk bilangan pecahan dengan presisi sempurna. Pembahasan mencakup perbandingan antara INTEGER, NUMERIC, dan floating-point, serta cara menggunakan parameter precision dan scale untuk mengontrol format angka. Tipe ini ideal untuk data finansial yang membutuhkan akurasi mutlak.

## 2. Konsep Utama

### a. Spektrum Tipe Data Numerik

Ada **tiga kategori utama** untuk menyimpan angka di PostgreSQL, masing-masing dengan trade-off berbeda:

**Visualisasi spektrum:**

```
[INTEGER] ←→ [NUMERIC/DECIMAL] ←→ [FLOATING-POINT]
  Fast         Slow, Precise       Fast
  Accurate     Accurate            Approximate
  Whole only   Fractional          Fractional
```

**Perbandingan detail:**

| Aspek                  | INTEGER                 | NUMERIC/DECIMAL     | FLOATING-POINT         |
| ---------------------- | ----------------------- | ------------------- | ---------------------- |
| **Support fractional** | ❌ Hanya bilangan bulat | ✅ Ya               | ✅ Ya                  |
| **Akurasi**            | ✅ Sempurna             | ✅ Sempurna         | ❌ Aproksimasi         |
| **Presisi**            | ✅ Tidak ada loss       | ✅ Tidak ada loss   | ❌ Bisa hilang presisi |
| **Kecepatan operasi**  | ⚡ Sangat cepat         | 🐌 Sangat lambat    | ⚡ Sangat cepat        |
| **Use case**           | Counter, ID, quantity   | **Uang, finansial** | Sains, grafik, approx. |

**Analogi sederhana:**

```
INTEGER        = Menghitung kelereng (1, 2, 3, 4...)
NUMERIC        = Menghitung uang (Rp 1.250,75) - harus tepat!
FLOATING-POINT = Mengukur suhu (23.456°C) - boleh sedikit meleset
```

### b. NUMERIC vs DECIMAL: Sama Persis

**Fakta penting:**

- `NUMERIC` dan `DECIMAL` adalah **identik** di PostgreSQL
- Keduanya adalah **alias** untuk tipe data yang sama
- Bisa digunakan bergantian tanpa perbedaan apa pun

**Bukti:**

```sql
-- Deklarasi dengan DECIMAL
CREATE TABLE test1 (
    interest_rate DECIMAL
);

-- Deklarasi dengan NUMERIC
CREATE TABLE test2 (
    interest_rate NUMERIC
);

-- Cek dengan \d di psql
\d test1
-- Column: interest_rate | Type: numeric

\d test2
-- Column: interest_rate | Type: numeric

-- Keduanya menunjukkan "numeric" - mereka sama!
```

**Preferensi pribadi:**

- Instruktor lebih suka `NUMERIC` (lebih jelas)
- Pilih salah satu dan **konsisten** dalam codebase Anda

### c. NUMERIC Tanpa Parameter: Unbounded Precision

**Karakteristik NUMERIC tanpa parameter:**

```sql
CREATE TABLE unlimited (
    value NUMERIC  -- Tanpa parameter apa pun
);
```

**Kemampuan:**

1. **Tidak ada batasan range** - bisa lebih besar dari BIGINT
2. **Presisi sempurna** - tidak ada loss precision
3. **Support decimal** - bisa menyimpan pecahan
4. **Seperti TEXT column** - menerima hampir semua input

**Contoh: Lebih besar dari BIGINT**

```sql
-- BIGINT max: ±9,223,372,036,854,775,807 (9 quintillion)

-- ❌ Angka terlalu besar untuk BIGINT
SELECT 92233720368547758079223372036854775807::BIGINT;
-- ERROR: bigint out of range

-- ✅ NUMERIC bisa handle angka sebesar ini
SELECT 92233720368547758079223372036854775807::NUMERIC;
-- Result: 92233720368547758079223372036854775807

-- ✅ NUMERIC juga bisa dengan decimal
SELECT 92233720368547758079223372036854775807.12345::NUMERIC;
-- Result: 92233720368547758079223372036854775807.12345
```

**Kapan menggunakan NUMERIC unbounded:**

- Ketika **tidak tahu** batas data di masa depan
- Data **finansial global** dengan berbagai jenis mata uang
- **Perhitungan ilmiah** yang butuh presisi arbitrary

**Trade-off:**

- ✅ **Pro:** Flexibility maksimal, tidak ada batasan
- ❌ **Con:** Sangat lambat, tidak ada validasi built-in

### d. NUMERIC dengan Parameter: Precision dan Scale

**Sintaks:**

```sql
NUMERIC(precision, scale)
```

**Definisi:**

- **Precision:** Total jumlah digit (keseluruhan angka)
- **Scale:** Jumlah digit di **sebelah kanan** decimal point

**Ilustrasi visual:**

```
Angka: 12.345

├─ Precision = 5 (total semua digit: 1, 2, 3, 4, 5)
└─ Scale     = 3 (digit setelah titik: 3, 4, 5)

Left side : 2 digits (1, 2)
Right side: 3 digits (3, 4, 5)
```

**Contoh implementasi:**

```sql
-- NUMERIC(5, 3) = 5 total digits, 3 di kanan decimal
CREATE TABLE measurements (
    value NUMERIC(5, 3)
);

-- ✅ VALID: Sesuai dengan (5, 3)
INSERT INTO measurements VALUES (12.345);
-- Result: 12.345
-- Total digits: 5 ✓
-- Digits after decimal: 3 ✓

-- ❌ ERROR: Terlalu banyak digit total
INSERT INTO measurements VALUES (123.45);
-- ERROR: numeric field overflow
-- Reason: 123.45 = 5 digits, tapi left side ada 3 (lebih dari 2)

-- ✅ VALID tapi ada rounding
INSERT INTO measurements VALUES (12.3456);
-- Result: 12.346 (rounded dari 12.3456)
-- Reason: Scale = 3, jadi digit ke-4 (6) di-round
```

### e. Parameter NUMERIC: Variasi Penggunaan

#### **1. Single Parameter (Precision Only)**

```sql
-- NUMERIC(5) = precision 5, scale default 0
CREATE TABLE whole_numbers (
    value NUMERIC(5)  -- Sama dengan NUMERIC(5, 0)
);

-- ✅ Valid (bilangan bulat)
INSERT INTO whole_numbers VALUES (12345);
-- Result: 12345

-- ✅ Valid dengan rounding
INSERT INTO whole_numbers VALUES (123.45);
-- Result: 123 (desimal dihilangkan)

-- Explicit sama dengan implicit
CREATE TABLE explicit (
    value NUMERIC(5, 0)  -- Sama persis dengan NUMERIC(5)
);
```

#### **2. Negative Scale (Rounding dari Decimal)**

**Fitur baru:** Tersedia sejak **PostgreSQL 15+**

Fitur ini memungkinkan `NUMERIC(p, s)` memiliki **scale negatif**, yang berarti pembulatan dilakukan **ke kiri titik desimal**, bukan ke kanan seperti biasanya.

```sql
-- NUMERIC(5, -2) = round 2 tempat dari kiri decimal point
CREATE TABLE rounded (
    value NUMERIC(5, -2)
);

-- Input: 1234567
-- Precision = 5 (significant digits sebelum rounding)
-- Scale = -2 (round 2 digit dari kanan)
INSERT INTO rounded VALUES (1234567);
-- Result: 1234600
-- Proses:
-- 1. Ambil 5 significant digits dari 1234567: 12345
-- 2. Round 2 posisi dari kanan: 12346
-- 3. Tambah padding zeros: 1234600
-- 4. Hasil 12346 <= precision (5) -> OK (nol tidak dihitung).

-- Contoh lain
INSERT INTO rounded VALUES (12345.67);
-- Result: 12300
-- Proses:
-- 1. Ambil 5 significant digits dari 12345: 12345
-- 2. Round 2 posisi dan tambah pading: 123 -> 12300
-- 3. Hasil 12300 <= precision (5) -> OK
```

**Penjelasan "significant digits":**

```
Input       : 1234567
Precision   : 5
Scale       : -2

Step 1: Identify 5 significant unrounded digits
        -> jumlah digit significant = 7 (1234567)

Step 2: PostgreSQL mencoba membulatkan 2 digit terakhir (67)
        -> 12345678 → dibulatkan menjadi 1234600

Step 3: Hasil pembulatan masih memiliki 5 digit signifikan (1234600)
        -> Kurang atau sama dengan precision (5) -> OK!
```

**PERHATIAN: Total output bisa > precision!**

- Precision mengacu pada **significant UNROUNDED digits**
- Zeros padding dari scale negatif **tidak dihitung dalam precision**

**Contoh Error: Overflow**

```sql
SELECT 12345678::NUMERIC(5, -2);
-- ERROR:  numeric field overflow
```

**Penjelasan:**

```
Input       : 12345678
Precision   : 5
Scale       : -2

Step 1: Cek jumlah digit signifikan (unrounded)
        -> jumlah digit siginifikan = 8 (12345678)

Step 2: PostgreSQL mencoba membulatkan 2 digit terakhir (78)
        -> 12345678 → dibulatkan menjadi 12345700

Step 3: Hasil pembulatan masih memiliki 6 digit signifikan (12345700)
        -> Melebihi precision (5) -> !OK

=> ERROR: numeric field overflow
```

**Intinya:**
Walaupun `scale = -2` hanya menambah nol di belakang, PostgreSQL tetap memeriksa **jumlah digit signifikan sebelum nol padding**.
Jika hasil pembulatan melebihi batas `precision`, maka PostgreSQL akan menolak nilai tersebut.

### f. Best Practices untuk NUMERIC

**1. Untuk data finansial (uang):**

```sql
-- ✅ RECOMMENDED: Always use NUMERIC for money
CREATE TABLE transactions (
    amount NUMERIC(10, 2)  -- Max 99,999,999.99
);

-- Contoh: IDR (Rupiah)
-- Max: 99,999,999.99 = ~100 juta rupiah
-- Precision cukup untuk transaksi retail

-- Contoh: USD (Dollar)
-- Max: 99,999,999.99 = ~100 juta dollar
-- Precision cukup untuk kebanyakan transaksi bisnis

-- Untuk transaksi lebih besar
CREATE TABLE large_transactions (
    amount NUMERIC(15, 2)  -- Max 9,999,999,999,999.99
);
```

**2. Untuk interest rates:**

```sql
-- Interest rate biasanya dalam persen dengan presisi tinggi
CREATE TABLE loans (
    description TEXT,
    interest_rate NUMERIC(5, 4)  -- Contoh: 5.2500%
);

-- Contoh values:
-- 5.2500 = 5.25%
-- 12.3456 = 12.3456%
-- 0.0125 = 0.0125% (untuk rate sangat kecil)

INSERT INTO loans VALUES
    ('Home Loan', 5.2500),
    ('Car Loan', 12.3456),
    ('Micro Finance', 0.0125);
```

**3. Untuk scientific calculations:**

```sql
-- Jika butuh presisi sempurna (rare)
CREATE TABLE scientific_data (
    measurement NUMERIC  -- Unbounded untuk flexibility
);

-- Tapi biasanya floating-point lebih cocok untuk sains
-- (akan dibahas video berikutnya)
```

**4. Hindari over-specification:**

```sql
-- ❌ OVERKILL: Precision terlalu ketat tanpa alasan
CREATE TABLE simple_prices (
    price NUMERIC(8, 2)  -- Max 999,999.99
);
-- Better: NUMERIC(10, 2) - sedikit lebih flexible

-- ❌ TOO LOOSE: Unbounded tanpa alasan
CREATE TABLE product_prices (
    price NUMERIC  -- Tidak ada validasi!
);
-- Better: NUMERIC(10, 2) - batasan jelas
```

### g. Demonstrasi Membuktikan Fakta (Meta-Learning)

**Poin penting dari video:**

> "Reading the docs is great, reading books is great, **proving something to yourself - honestly, that's better**."

**Metode yang ditunjukkan:**

**1. Menggunakan `\d` (describe) di psql:**

```bash
-- Cek struktur table
\d table_name

-- Verify tipe data yang sebenarnya
CREATE TABLE test (value DECIMAL);
\d test
-- Output: Type: numeric (bukan decimal!)
-- Bukti: DECIMAL = alias dari NUMERIC
```

**2. Menggunakan `CAST` untuk testing:**

```sql
-- Test batasan BIGINT
SELECT 92233720368547758079223372036854775807::BIGINT;
-- ERROR: Membuktikan BIGINT punya batasan

-- Test kemampuan NUMERIC
SELECT 92233720368547758079223372036854775807::NUMERIC;
-- SUCCESS: Membuktikan NUMERIC bisa lebih besar
```

**3. Trial and error dengan INSERT:**

```sql
-- Test precision dan scale
CREATE TABLE test (value NUMERIC(5, 3));

INSERT INTO test VALUES (12.345);   -- ✅ Berhasil
INSERT INTO test VALUES (123.45);   -- ❌ Error
INSERT INTO test VALUES (12.3456);  -- ✅ Rounded
```

**Takeaway:**

- Jangan hanya **percaya dokumentasi** (verify!)
- Jangan hanya **percaya tutorial** (test sendiri!)
- **Hands-on experimentation** = pembelajaran terbaik

## 3. Hubungan Antar Konsep

**Progression dari INTEGER ke NUMERIC:**

```
INTEGER (Video sebelumnya)
    ↓
    ├─ Fast, accurate, tapi hanya whole numbers
    └─ Kapan butuh fractional numbers?
        ↓
    NUMERIC (Video ini)
        ↓
        ├─ Slow tapi perfectly precise
        ├─ Biasa untuk financial data
        └─ Parameter (precision, scale) untuk control
            ↓
            ├─ Unbounded: Flexibility maksimal
            ├─ Bounded: Validation built-in
            └─ Negative scale: Advanced rounding
```

**Decision tree memilih tipe:**

```
Apakah data berupa angka?
├─ Ya → Lanjut
└─ Tidak → TEXT/VARCHAR

Apakah butuh fractional numbers?
├─ Tidak → INTEGER (SMALLINT/INT/BIGINT)
└─ Ya → Lanjut

Apakah akurasi sempurna MUTLAK diperlukan?
├─ Ya → NUMERIC
└─ Tidak (boleh aproksimasi) → FLOATING-POINT (next video)
```

## 4. Kesimpulan

`NUMERIC` (alias `DECIMAL`) adalah tipe data PostgreSQL untuk bilangan pecahan dengan **presisi sempurna**. Berada di tengah spektrum antara INTEGER (cepat, whole numbers only) dan FLOATING-POINT (cepat, tapi approximate).

**Key points:**

1. **NUMERIC = DECIMAL:** Keduanya identik, hanya alias

2. **Perfectly accurate:** Tidak ada loss precision seperti FLOAT - **wajib untuk financial data**

3. **Trade-off:** Sangat lambat dibanding INTEGER dan FLOAT

4. **Unbounded by default:** `NUMERIC` tanpa parameter bisa lebih besar dari BIGINT

5. **Precision & Scale untuk control:**

   - `NUMERIC(10, 2)`: 10 total digits, 2 setelah decimal
   - Pattern umum untuk uang: `(10, 2)` atau `(15, 2)`
   - Pattern untuk rates: `(5, 4)` atau `(6, 4)`

6. **Negative scale:** Rounding dari kiri decimal point (PostgreSQL 15+)

7. **Best practice:**
   - Use NUMERIC untuk money intead of FLOAT
   - Specify precision untuk production (validation)
   - Don't over-optimize (gunakan reasonable bounds)

**Kapan menggunakan NUMERIC:**

- ✅ Financial data (harga, salary, tax)
- ✅ Interest rates, percentages yang presisi
- ✅ Accounting, billing systems
- ❌ Counters, quantities (gunakan INTEGER)
- ❌ Scientific approximations (gunakan FLOAT - next video)

NUMERIC memberikan **peace of mind** untuk financial applications - presisi sempurna tanpa rounding errors yang bisa menyebabkan discrepancies dalam accounting. Trade-off performance adalah harga yang layak dibayar untuk akurasi finansial.
