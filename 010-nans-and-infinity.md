# 📝 NaN dan Infinity dalam PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas beberapa keunikan dari tipe data numerik di PostgreSQL, khususnya tentang nilai-nilai spesial seperti NaN (Not a Number) dan Infinity. Materi ini menjelaskan tipe data mana yang mendukung nilai-nilai tersebut, bagaimana perilaku mereka dalam operasi matematika, dan kapan nilai-nilai ini mungkin berguna dalam praktik.

## 2. Konsep Utama

### a. NaN (Not a Number)

NaN adalah singkatan dari "Not a Number" yang merepresentasikan nilai yang bukan merupakan angka valid.

**Karakteristik NaN:**

- **Tipe data yang mendukung**: Hanya `NUMERIC` dan `FLOAT` yang memiliki NaN
- **Tipe data yang tidak mendukung**: `INTEGER` tidak memiliki NaN
- **Kesetaraan**: Di PostgreSQL, semua NaN dianggap sama dengan NaN lainnya (ini adalah perilaku spesifik PostgreSQL)
- **Dalam sorting**: NaN dianggap sebagai nilai yang sangat besar, lebih besar dari angka manapun

**Contoh:**

```sql
-- Membuat nilai NaN
SELECT 'NaN'::NUMERIC;  -- Menghasilkan NaN

-- NaN lebih besar dari angka manapun dalam sorting
SELECT 'NaN'::NUMERIC > 999999999::NUMERIC;  -- TRUE

-- Operasi dengan NaN
SELECT 'NaN'::NUMERIC + 1;  -- Menghasilkan NaN
SELECT POWER('NaN'::NUMERIC, 0);  -- Menghasilkan 1 (karena x^0 = 1)
```

### b. Infinity (Nilai Tak Terbatas)

Infinity merepresentasikan nilai yang tidak terbatas (unbounded), baik positif maupun negatif.

**Karakteristik Infinity:**

- **Tipe data yang mendukung**:
  - `NUMERIC` tanpa precision dan scale (open-ended numeric)
  - `FLOAT`
- **Tipe data yang tidak mendukung**:
  - `INTEGER` (karena integer secara definisi memiliki batas)
  - `NUMERIC` dengan range yang ditentukan (closed numeric)
- **Varian**: Ada `infinity` dan `-infinity` (negative infinity)
- **Kesetaraan**: Semua infinity sama dengan infinity lainnya

**Contoh:**

```sql
-- Membuat nilai infinity
SELECT 'infinity'::NUMERIC;
SELECT '-infinity'::NUMERIC;

-- Operasi matematika dengan infinity
SELECT 'infinity'::NUMERIC + 'infinity'::NUMERIC;  -- infinity
SELECT 'infinity'::NUMERIC - 'infinity'::NUMERIC;  -- NaN
SELECT 'infinity'::NUMERIC + 1;  -- infinity
SELECT 'infinity'::NUMERIC - 1;  -- infinity
```

### c. Perilaku dalam Operasi Matematika

Operasi matematika dengan NaN dan Infinity memiliki aturan khusus:

**Operasi Infinity:**

- `infinity + infinity = infinity`
- `infinity - infinity = NaN` (tidak terdefinisi)
- `infinity + 1 = infinity`
- `infinity - 1 = infinity`

**Operasi NaN:**

- `NaN + angka_apapun = NaN`
- `NaN ^ 0 = 1` (pengecualian: karena aturan matematika x⁰ = 1)

**Contoh:**

```sql
-- Infinity
SELECT 'infinity'::NUMERIC + 'infinity'::NUMERIC;  -- infinity
SELECT 'infinity'::NUMERIC - 'infinity'::NUMERIC;  -- NaN (filosofis/matematis)

-- NaN
SELECT 'NaN'::NUMERIC + 1;  -- NaN
SELECT POWER('NaN'::NUMERIC, 0);  -- 1
```

## 3. Hubungan Antar Konsep

- **NaN dan Infinity adalah nilai spesial** yang hanya tersedia pada tipe data numerik yang tidak terbatas (unbounded)
- **INTEGER tidak mendukung keduanya** karena integer secara definisi memiliki batas minimum dan maksimum
- **NUMERIC dengan precision dan scale** juga tidak mendukung infinity karena range-nya sudah ditentukan
- **Dalam operasi matematika**, NaN cenderung "menginfeksi" hasil perhitungan (operasi apapun dengan NaN menghasilkan NaN), kecuali kasus khusus seperti pangkat nol
- **Infinity lebih stabil** dalam operasi matematika dan mengikuti logika matematika (infinity ± bilangan finite tetap infinity)

## 4. Catatan Tambahan / Insight

### Tips Praktis:

1. **Kapan menggunakan Infinity:**

   - Saat membutuhkan nilai yang sangat besar atau sangat kecil tanpa batas pasti
   - Sebagai placeholder untuk "tidak terbatas" dalam model data
   - Dalam constraint atau validasi untuk mewakili "tanpa batas atas/bawah"

2. **Kapan menggunakan NaN:**

   - Use case untuk NaN kurang jelas dalam praktik
   - Lebih sering muncul sebagai hasil dari operasi yang invalid (seperti `infinity - infinity`)
   - Bisa berguna untuk menandai perhitungan yang gagal tanpa menggunakan NULL

3. **Perbedaan dengan NULL:**

   - NULL berarti "tidak ada nilai"
   - NaN berarti "nilai ada, tetapi bukan angka valid"
   - Infinity berarti "nilai ada dan sangat besar/kecil tanpa batas"

4. **Sorting behavior:**
   - NaN dianggap lebih besar dari nilai manapun termasuk infinity
   - Order: `-infinity < ... angka ... < infinity < NaN`

### Analogi:

- **NaN** seperti hasil dari pembagian 0/0 - secara matematis tidak terdefinisi
- **Infinity** seperti garis yang tidak pernah berakhir - konsep yang valid secara matematis
- **Integer** seperti kotak dengan ukuran tetap - tidak bisa menampung "tidak terbatas"

## 5. Kesimpulan

NaN dan Infinity adalah nilai-nilai spesial dalam PostgreSQL yang hanya tersedia pada tipe data `NUMERIC` (tanpa precision/scale) dan `FLOAT`. Nilai-nilai ini tidak tersedia pada `INTEGER` atau `NUMERIC` dengan range yang ditentukan.

Infinity lebih berguna dalam praktik untuk merepresentasikan nilai yang tidak terbatas, sementara NaN lebih sering muncul sebagai hasil operasi yang tidak valid. Keduanya memiliki perilaku khusus dalam operasi matematika dan sorting yang penting untuk dipahami saat bekerja dengan data numerik di PostgreSQL.

**Key takeaway:** Pahami batasan tipe data yang digunakan - hanya unbounded numeric types yang mendukung NaN dan Infinity.
