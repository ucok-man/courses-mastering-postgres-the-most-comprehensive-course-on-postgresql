# 📝 Range Types di PostgreSQL: Menyimpan dan Query Data Rentang

## 1. Ringkasan Singkat

Video ini membahas **range types** di PostgreSQL, yaitu tipe data yang merepresentasikan rentang nilai dengan **lower bound** dan **upper bound**. Range adalah meta type yang bisa memiliki berbagai subtype (integer, numeric, date, timestamp). Pembahasan mencakup cara membuat range, perbedaan antara **inclusive/exclusive bounds**, **continuous vs discrete ranges**, operasi query (containment, overlap, intersection), serta pengenalan **multi-range** untuk rentang non-contiguous.

---

## 2. Konsep Utama

### a. Apa itu Range Types?

**Definisi:**
Range adalah **meta type** yang merepresentasikan rentang nilai dengan batasan bawah (lower bound) dan batasan atas (upper bound).

**Karakteristik:**

- Bukan tipe data standalone, tapi **meta type dengan subtype**
- Setiap range memiliki **subtype** seperti integer, numeric, date, timestamp
- Bounds bisa **inclusive** (termasuk nilai) atau **exclusive** (tidak termasuk nilai)
- Bounds bisa **unbounded** (tanpa batas atas/bawah)

**Jenis-jenis range types:**

- `INT4RANGE` - range untuk integer 4-byte
- `INT8RANGE` - range untuk integer 8-byte (bigint)
- `NUMRANGE` - range untuk numeric (continuous)
- `DATERANGE` - range untuk date (discrete)
- `TSRANGE` - range untuk timestamp without timezone
- `TSTZRANGE` - range untuk timestamp with timezone

---

### b. Cara Membuat Range

**Metode 1: String literal dengan casting**

```sql
-- Format: '[lower,upper]'::rangetype
SELECT '[1,5]'::INT4RANGE;
-- Output: [1,6) -- PostgreSQL mengkonversi ke format standard

SELECT '[1,5]'::NUMRANGE;
-- Output: [1,5] -- numeric range tetap seperti input
```

**Metode 2: Constructor functions**

```sql
-- Format: rangetype(lower, upper)
-- Default: lower inclusive, upper exclusive
SELECT NUMRANGE(1, 5);
-- Output: [1,5)

SELECT INT4RANGE(1, 5);
-- Output: [1,5)

-- Dengan parameter ketiga untuk menentukan bounds
SELECT NUMRANGE(1, 5, '[]');  -- kedua inclusive
-- Output: [1,5]

SELECT NUMRANGE(1, 5, '()');  -- kedua exclusive
-- Output: (1,5)

SELECT NUMRANGE(1, 5, '(]');  -- lower exclusive, upper inclusive
-- Output: (1,5]

SELECT NUMRANGE(1, 5, '[)');  -- lower inclusive, upper exclusive (default)
-- Output: [1,5)
```

**Notasi bounds:**

- `[` atau `]` = **inclusive** (nilai termasuk dalam range)
- `(` atau `)` = **exclusive** (nilai tidak termasuk dalam range)

---

### c. Discrete vs Continuous Ranges

**Perbedaan fundamental:**

**Discrete Range (memiliki step diskrit):**

- Contoh: `INT4RANGE`, `INT8RANGE`, `DATERANGE`
- Ada **unit step** yang jelas (1, 2, 3 untuk integer; hari untuk date)
- PostgreSQL akan **mengkonversi** bounds untuk konsistensi

```sql
-- Discrete range: [1,5] dikonversi jadi [1,6)
SELECT '[1,5]'::INT4RANGE;
-- Output: [1,6)
-- Alasan: 1,2,3,4,5 bisa ditulis sebagai [1,5] atau [1,6)
```

**Continuous Range (tanpa step diskrit):**

- Contoh: `NUMRANGE`, `TSRANGE`, `TSTZRANGE`
- Ada **infinite values** di antara dua angka
- PostgreSQL **tidak mengkonversi** bounds

```sql
-- Continuous range: [1,5] tetap [1,5]
SELECT '[1,5]'::NUMRANGE;
-- Output: [1,5]
-- Alasan: 5.99, 5.999, 5.9999... berbeda dengan 6

-- [1,5] TIDAK SAMA dengan [1,6) untuk continuous
SELECT '[1,6)'::NUMRANGE;
-- Output: [1,6)
-- Nilai 5.99 valid di sini, tapi tidak di [1,5]
```

**Diagram perbandingan:**

```
Discrete Range (INT4RANGE):
[1,5]  ≡  [1,6)
└─ nilai valid: 1, 2, 3, 4, 5

Continuous Range (NUMRANGE):
[1,5]  ≠  [1,6)
└─ [1,5]: valid sampai 5.0
└─ [1,6): valid sampai 5.999...
```

---

### d. Unbounded dan Empty Ranges

**Unbounded Range:**
Range tanpa batas atas atau bawah.

```sql
-- Unbounded upper: dari 5 ke infinity
SELECT '[5,)'::INT4RANGE;
-- Output: [5,)

-- Unbounded lower: dari -infinity sampai 10
SELECT '(,10]'::INT4RANGE;
-- Output: (,10]

-- Unbounded both: seluruh rentang waktu
SELECT '(,)'::DATERANGE;
-- Output: (,)
```

**Use case unbounded:**

- Hotel room "under construction" forever
- Product "discontinued" mulai tanggal tertentu tanpa akhir

**Empty Range:**
Range yang **tidak merepresentasikan nilai apapun**.

```sql
SELECT 'empty'::INT4RANGE;
-- Output: empty
```

**Perbedaan empty vs unbounded:**

- `empty` = **tidak ada rentang sama sekali**
- `(,)` = **seluruh rentang yang mungkin**
- Keduanya **fundamentally berbeda**!

---

### e. Operator Containment (`@>`)

Operator untuk mengecek apakah **range mengandung nilai tertentu**.

**Sintaks:**

```sql
range @> value
```

**Contoh:**

```sql
-- Setup table
CREATE TABLE range_example (
  id SERIAL,
  int_range INT4RANGE
);

INSERT INTO range_example (int_range) VALUES
  ('[1,11)'::INT4RANGE),
  ('[2,101)'::INT4RANGE),
  ('[5,)'::INT4RANGE);

-- Cari range yang mengandung nilai 5
SELECT * FROM range_example
WHERE int_range @> 5;
-- Output: semua rows di atas (5 ada di semua range)

-- Cari range yang mengandung nilai 11
SELECT * FROM range_example
WHERE int_range @> 11;
-- Output: hanya row 2 dan 3
-- (row 1 adalah [1,11) yang exclude 11)
```

**Perhatikan pengaruh inclusive/exclusive:**

```sql
-- Range [5,) dengan 5 inclusive
SELECT '[5,)'::INT4RANGE @> 5;
-- Output: true

-- Range (5,) dengan 5 exclusive
SELECT '(5,)'::INT4RANGE @> 5;
-- Output: false
```

---

### f. Operator Overlap (`&&`)

Operator untuk mengecek apakah **dua range tumpang tindih**.

**Sintaks:**

```sql
range1 && range2
```

**Contoh:**

```sql
SELECT id, int_range
FROM range_example
WHERE int_range && '[10,20)'::INT4RANGE;
```

**Hasil:**

| id  | int_range | Alasan overlap               |
| --- | --------- | ---------------------------- |
| 1   | [1,11)    | overlap di 10                |
| 2   | [2,101)   | overlap di 10-19             |
| 3   | [5,)      | overlap di 10-19 (unbounded) |

**Contoh edge case:**

```sql
-- Apakah [1,11) overlap dengan [11,20)?
SELECT '[1,11)'::INT4RANGE && '[11,20)'::INT4RANGE;
-- Output: false
-- Alasan: 11 excluded di range pertama, included di range kedua
-- Tidak ada nilai yang sama-sama termasuk

-- Apakah [1,11] overlap dengan [11,20)?
SELECT '[1,11]'::INT4RANGE && '[11,20)'::INT4RANGE;
-- Output: true
-- Alasan: nilai 11 ada di kedua range
```

---

### g. Operator Intersection (`*`)

Operator untuk **menghitung irisan** dua range.

**Sintaks:**

```sql
range1 * range2
```

**Contoh:**

```sql
-- Irisan [10,20) dan [15,25)
SELECT INT4RANGE(10, 20) * INT4RANGE(15, 25);
-- Output: [15,20)
-- Alasan: nilai yang ada di kedua range adalah 15-19

-- Irisan dengan inclusive bound
SELECT INT4RANGE(10, 20, '[]') * INT4RANGE(15, 25);
-- Output: [15,21)
-- Untuk discrete range, [15,20] ≡ [15,21)
```

**Diagram intersection:**

```
Range 1:  [====10---15---20========)
Range 2:         [===15---20---25==)
                      ↓
Intersection:    [===15---20===)
```

---

### h. Fungsi Upper dan Lower

Fungsi untuk mendapatkan **batas atas dan bawah** range secara programatis.

**Fungsi tersedia:**

- `LOWER(range)` - ambil lower bound
- `UPPER(range)` - ambil upper bound
- `LOWER_INC(range)` - cek apakah lower bound inclusive
- `UPPER_INC(range)` - cek apakah upper bound inclusive

**Contoh dengan discrete range:**

```sql
SELECT
  UPPER('[10,20]'::INT4RANGE) AS upper_val,
  UPPER_INC('[10,20]'::INT4RANGE) AS upper_inclusive;
-- Output: upper_val=21, upper_inclusive=false
```

**Mengapa upper_val adalah 21?**

- Input: `[10,20]` (20 inclusive)
- Untuk discrete range: `[10,20]` ≡ `[10,21)`
- `UPPER()` selalu return dalam format exclusive
- Maka: upper = 21, upper_inclusive = false

**Contoh dengan continuous range:**

```sql
SELECT
  UPPER('[10,20)'::NUMRANGE) AS upper_val,
  UPPER_INC('[10,20)'::NUMRANGE) AS upper_inclusive;
-- Output: upper_val=20, upper_inclusive=false
```

**Mengapa ini satu-satunya cara?**
Untuk continuous range dengan upper exclusive, **tidak ada cara lain** merepresentasikan batas atas:

- Tidak bisa bilang 19.999... (infinite 9s)
- Hanya bisa bilang "20 excluded"

**Contoh lengkap:**

```sql
SELECT
  LOWER('[10,20]'::INT4RANGE) AS lower_val,
  LOWER_INC('[10,20]'::INT4RANGE) AS lower_inclusive,
  UPPER('[10,20]'::INT4RANGE) AS upper_val,
  UPPER_INC('[10,20]'::INT4RANGE) AS upper_inclusive;
-- Output:
-- lower_val=10, lower_inclusive=true
-- upper_val=21, upper_inclusive=false
```

---

### i. Multi-Range (Non-Contiguous Ranges)

**Definisi:**
Multi-range adalah tipe data yang bisa menyimpan **beberapa range sekaligus** untuk membentuk rentang **non-contiguous** (tidak bersebelahan).

**Jenis multi-range:**

- `INT4MULTIRANGE`
- `INT8MULTIRANGE`
- `NUMMULTIRANGE`
- `DATEMULTIRANGE`
- `TSMULTIRANGE`
- `TSTZMULTIRANGE`

**Contoh:**

```sql
-- Multi-range: 3-6 dan 8
SELECT '{[3,7), [8,9)}'::INT4MULTIRANGE;
-- Output: {[3,7),[8,9)}
-- Merepresentasikan nilai: 3,4,5,6,8
```

**Operator containment pada multi-range:**

```sql
-- Cek apakah 4 ada dalam multi-range
SELECT '{[3,7), [8,9)}'::INT4MULTIRANGE @> 4;
-- Output: true (4 ada di range [3,7))

-- Cek apakah 7 ada dalam multi-range
SELECT '{[3,7), [8,9)}'::INT4MULTIRANGE @> 7;
-- Output: false (7 excluded dari [3,7))

-- Cek apakah 8 ada dalam multi-range
SELECT '{[3,7), [8,9)}'::INT4MULTIRANGE @> 8;
-- Output: true (8 ada di range [8,9))

-- Cek apakah 9 ada dalam multi-range
SELECT '{[3,7), [8,9)}'::INT4MULTIRANGE @> 9;
-- Output: false (9 excluded dari [8,9))
```

**Use case multi-range:**

- **Business hours**: Pagi 08:00-12:00, Sore 13:00-17:00
- **Availability**: Available hari Senin-Rabu dan Jumat-Sabtu
- **Discount periods**: Diskon aktif bulan Jan-Feb dan Jun-Aug

**Diagram multi-range:**

```
Single Range:     [========3---4---5---6---7========)
                                     GAP!
Multi-Range:      [====3---6====)   [==8==)
                   ↑ range 1 ↑       ↑ range 2 ↑
```

---

### j. Tabel Lengkap Range Types

**Range types yang tersedia:**

| Range Type  | Multi-Range Type | Subtype             | Category   |
| ----------- | ---------------- | ------------------- | ---------- |
| `INT4RANGE` | `INT4MULTIRANGE` | `integer`           | Discrete   |
| `INT8RANGE` | `INT8MULTIRANGE` | `bigint`            | Discrete   |
| `NUMRANGE`  | `NUMMULTIRANGE`  | `numeric`           | Continuous |
| `TSRANGE`   | `TSMULTIRANGE`   | `timestamp`         | Continuous |
| `TSTZRANGE` | `TSTZMULTIRANGE` | `timestamp with tz` | Continuous |
| `DATERANGE` | `DATEMULTIRANGE` | `date`              | Discrete   |

---

### k. Use Case: Hotel Room Reservations

**Problem statement:**
Hotel perlu memastikan **tidak ada reservasi yang overlap** untuk room yang sama.

**Solusi dengan range + exclusion constraint:**

```sql
CREATE TABLE reservations (
  id SERIAL PRIMARY KEY,
  room_id INT,
  guest_name TEXT,
  stay_period DATERANGE,
  -- Exclusion constraint: cegah overlap untuk room yang sama
  EXCLUDE USING GIST (
    room_id WITH =,
    stay_period WITH &&
  )
);

-- Insert reservasi pertama: OK
INSERT INTO reservations (room_id, guest_name, stay_period)
VALUES (101, 'Alice', '[2025-01-01,2025-01-05)'::DATERANGE);

-- Insert reservasi overlap: ERROR
INSERT INTO reservations (room_id, guest_name, stay_period)
VALUES (101, 'Bob', '[2025-01-03,2025-01-07)'::DATERANGE);
-- ERROR: conflicting key value violates exclusion constraint

-- Insert reservasi berbeda room: OK
INSERT INTO reservations (room_id, guest_name, stay_period)
VALUES (102, 'Bob', '[2025-01-03,2025-01-07)'::DATERANGE);

-- Insert reservasi setelah check-out: OK
INSERT INTO reservations (room_id, guest_name, stay_period)
VALUES (101, 'Charlie', '[2025-01-05,2025-01-10)'::DATERANGE);
```

**Penjelasan exclusion constraint:**

- `room_id WITH =`: room_id harus sama
- `stay_period WITH &&`: DAN period overlap
- Jika kedua kondisi terpenuhi → **constraint violation**

---

## 3. Hubungan Antar Konsep

```
┌──────────────────────────────────────────────┐
│         RANGE META TYPE                      │
└──────────────────────────────────────────────┘
                    │
        ┌───────────┴────────────┐
        │                        │
   ┌────▼────┐              ┌────▼─────┐
   │ Discrete│              │Continuous│
   │ Ranges  │              │ Ranges   │
   └────┬────┘              └────┬─────┘
        │                        │
   - INT4RANGE              - NUMRANGE
   - INT8RANGE              - TSRANGE
   - DATERANGE              - TSTZRANGE
        │                        │
        └───────────┬────────────┘
                    │
        ┌───────────▼────────────┐
        │   BOUNDS (inclusive/   │
        │   exclusive, unbounded)│
        └───────────┬────────────┘
                    │
        ┌───────────┴────────────┐
        │                        │
   ┌────▼────┐              ┌────▼──────┐
   │ Single  │              │Multi-Range│
   │ Range   │              │(multiple  │
   └────┬────┘              │ ranges)   │
        │                   └────┬──────┘
        │                        │
        └───────────┬────────────┘
                    │
        ┌───────────▼────────────┐
        │    OPERATORS           │
        ├────────────────────────┤
        │ @>  (containment)      │
        │ &&  (overlap)          │
        │ *   (intersection)     │
        │ UPPER/LOWER functions  │
        └────────────────────────┘
```

**Alur pemahaman:**

1. **Range adalah meta type** dengan berbagai subtype (int, numeric, date, timestamp)

2. **Discrete vs Continuous**:

   - Discrete: ada step unit → PostgreSQL normalize bounds
   - Continuous: infinite values → bounds tidak di-normalize

3. **Bounds menentukan inklusi**:

   - `[` `]` = inclusive (nilai termasuk)
   - `(` `)` = exclusive (nilai tidak termasuk)
   - Bisa unbounded (tanpa batas)
   - Bisa empty (tidak ada range)

4. **Single range vs Multi-range**:

   - Single: satu rentang contiguous
   - Multi: beberapa rentang non-contiguous

5. **Operators untuk query**:

   - Containment: apakah nilai ada dalam range?
   - Overlap: apakah dua range tumpang tindih?
   - Intersection: irisan dua range
   - Upper/Lower: ambil bounds programmatically

6. **Use case praktis**:
   - Reservasi hotel (cegah overlap)
   - Price tiers berdasarkan quantity
   - Access control based on time periods
   - Historical data versioning

---

## 4. Kesimpulan

Range types adalah fitur powerful PostgreSQL untuk merepresentasikan dan query data **rentang nilai**. Dengan range, kita bisa:

**Key features:**

- **Flexible bounds**: inclusive, exclusive, atau unbounded
- **Multiple subtypes**: integer, numeric, date, timestamp
- **Powerful operators**: containment, overlap, intersection
- **Multi-range support**: untuk rentang non-contiguous
- **Constraint enforcement**: cegah overlap dengan exclusion constraints

**Kapan menggunakan ranges:**

- **Reservasi/booking systems**: hotel, meeting room, equipment rental
- **Temporal data**: historical records, validity periods
- **Pricing tiers**: quantity discounts, membership levels
- **Access control**: time-based permissions, blackout periods

**Perbedaan penting:**

- **Discrete ranges** (int, date): PostgreSQL normalize bounds untuk konsistensi
- **Continuous ranges** (numeric, timestamp): bounds tetap seperti input
- **Empty** ≠ **unbounded**: empty = tidak ada range, unbounded = infinite range

**Complexity level:**

```
Basic:        Single range dengan fixed bounds
Intermediate: Unbounded ranges, overlap detection
Advanced:     Multi-ranges, exclusion constraints
```

Range types mungkin terlihat kompleks di awal, tapi setelah memahami konsep **discrete vs continuous** dan **inclusive vs exclusive**, semuanya akan jadi lebih jelas. Practice dengan berbagai contoh akan sangat membantu!
