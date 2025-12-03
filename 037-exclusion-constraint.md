# 📝 EXCLUSION Constraint di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas EXCLUSION constraint di PostgreSQL, sebuah constraint yang lebih powerful dan fleksibel dibandingkan UNIQUE constraint. EXCLUSION constraint memungkinkan kita untuk mencegah konflik data berdasarkan kriteria custom yang kompleks, seperti mencegah overlapping time ranges. Materi ini mencakup penggunaan GIST index, operator overlap, composite exclusion, extension btree_gist, dan partial exclusion constraint.

## 2. Konsep Utama

### a. Apa itu EXCLUSION Constraint?

**Definisi:**

- EXCLUSION constraint mirip dengan UNIQUE, tetapi lebih powerful dan fleksibel
- Memungkinkan kita mendefinisikan kondisi custom untuk mencegah konflik data
- Tidak sesering digunakan seperti UNIQUE, tapi sangat berguna untuk kasus tertentu

**Perbandingan dengan UNIQUE:**

```sql
-- UNIQUE constraint: Mencegah duplikasi nilai exact
CREATE TABLE users (
    email TEXT UNIQUE  -- Tidak boleh ada 2 email yang sama persis
);

-- EXCLUSION constraint: Mencegah konflik berdasarkan kondisi custom
CREATE TABLE reservations (
    reservation_period TSRANGE,
    EXCLUDE USING gist (reservation_period WITH &&)
    -- Tidak boleh ada 2 reservasi dengan periode yang overlap
);
```

**Kapan menggunakan EXCLUSION:**

- Ketika perlu mencegah overlapping ranges (waktu, tanggal, koordinat)
- Ketika konflik ditentukan oleh operator selain `=` (equality)
- Ketika butuh logika lebih kompleks dari sekedar "nilai harus berbeda"

### b. Struktur Dasar EXCLUSION Constraint

**Sintaks:**

```sql
EXCLUDE USING index_method (
    column_name WITH operator,
    ...
)
```

**Komponen:**

1. **EXCLUDE**: Kata kunci untuk membuat exclusion constraint
2. **USING gist**: Tipe index yang digunakan (akan dibahas di modul indexing)
3. **column_name**: Kolom yang akan dicek
4. **WITH operator**: Operator untuk menentukan konflik

**Contoh paling sederhana:**

```sql
CREATE TABLE reservations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INTEGER,
    reservation_period TSRANGE,
    EXCLUDE USING gist (reservation_period WITH &&)
);
```

**Penjelasan:**

- `reservation_period WITH &&`: Gunakan operator overlap (`&&`) untuk cek konflik
- Ketika insert/update row baru, database akan:
  1. Bandingkan `reservation_period` baru dengan semua row yang ada
  2. Gunakan operator `&&` (overlap) untuk cek konflik
  3. Jika ada overlap, tolak operasi tersebut

### c. Operator Overlap (`&&`) untuk Range Types

**Apa itu operator `&&`?**

- Operator untuk mengecek apakah dua range saling overlap
- Digunakan untuk range types seperti TSRANGE, DATERANGE, INT4RANGE, dll.

**Contoh penggunaan:**

```sql
-- Insert pertama berhasil
INSERT INTO reservations (room_id, reservation_period)
VALUES (1, '[2024-09-01 12:00, 2024-09-03 12:00)');
-- ✅ Berhasil

-- Data yang ada: 01 Sep - 03 Sep
-- Coba insert: 02 Sep - 04 Sep
INSERT INTO reservations (room_id, reservation_period)
VALUES (1, '[2024-09-02 12:00, 2024-09-04 12:00)');
-- ❌ Error: conflicting key value excludes reservation
```

**Visualisasi overlap:**

```
Existing: [01 Sep -------- 03 Sep)
New:              [02 Sep -------- 04 Sep)
                   ↑ Overlap di sini!
```

**Catatan penting tentang range bounds:**

- `[2024-09-01 12:00, 2024-09-03 12:00)`:
  - `[` = inclusive (termasuk 01 Sep 12:00)
  - `)` = exclusive (TIDAK termasuk 03 Sep 12:00, checkout harus sebelum jam ini)

### d. Masalah dengan EXCLUSION Sederhana

**Problem: Terlalu ketat untuk multi-room scenario**

```sql
CREATE TABLE reservations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INTEGER,
    reservation_period TSRANGE,
    EXCLUDE USING gist (reservation_period WITH &&)
);

-- Room 1: 01-03 Sep
INSERT INTO reservations VALUES (DEFAULT, 1, '[2024-09-01, 2024-09-03)');
-- ✅ Berhasil

-- Room 2: 01-03 Sep (harusnya boleh, karena room berbeda!)
INSERT INTO reservations VALUES (DEFAULT, 2, '[2024-09-01, 2024-09-03)');
-- ❌ Error: Ditolak! (padahal room berbeda)
```

**Mengapa ini masalah?**

- Constraint hanya memeriksa `reservation_period`
- Tidak memperhitungkan bahwa room berbeda boleh memiliki reservasi di waktu yang sama
- Solusi: Tambahkan `room_id` ke dalam exclusion constraint

### e. Composite EXCLUSION Constraint

**Solusi: Periksa kombinasi room_id DAN reservation_period**

```sql
CREATE TABLE reservations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INTEGER,
    reservation_period TSRANGE,
    EXCLUDE USING gist (
        room_id WITH =,              -- Room harus sama DAN
        reservation_period WITH &&   -- Periode overlap
    )
);
```

**Logika:**

- Row baru ditolak jika **KEDUA kondisi terpenuhi:**
  1. `room_id` sama (`=`)
  2. `reservation_period` overlap (`&&`)
- Jika room berbeda, walaupun periode sama, insert tetap boleh

**Contoh penggunaan:**

```sql
-- Room 1: 02-04 Sep
INSERT INTO reservations VALUES (DEFAULT, 1, '[2024-09-02, 2024-09-04)');
-- ✅ Berhasil

-- Room 2: 02-04 Sep (room berbeda, boleh!)
INSERT INTO reservations VALUES (DEFAULT, 2, '[2024-09-02, 2024-09-04)');
-- ✅ Berhasil

-- Room 2: 03-05 Sep (room sama dengan row sebelumnya, overlap!)
INSERT INTO reservations VALUES (DEFAULT, 2, '[2024-09-03, 2024-09-05)');
-- ❌ Error: Key (room_id, reservation_period)=(2, ...) conflicts with existing key
```

**Visualisasi:**

```
Room 1: [02 Sep -------- 04 Sep)  ✅
Room 2: [02 Sep -------- 04 Sep)  ✅ OK karena room berbeda
Room 2:        [03 Sep -------- 05 Sep)  ❌ Conflict! (room sama + overlap)
```

### f. Extension btree_gist

**Problem dengan operator `=` pada GIST index:**

```sql
EXCLUDE USING gist (
    room_id WITH =,  -- ❌ Error!
    reservation_period WITH &&
)
```

**Error message:**

```
ERROR: data type integer has no default operator class for access method "gist"
```

**Penjelasan:**

- GIST index tidak memiliki operator default untuk strict equality (`=`)
- GIST dirancang untuk data spasial dan range, bukan equality
- Btree index bagus untuk equality, GIST bagus untuk ranges

**Solusi: Enable extension btree_gist**

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- Sekarang constraint bisa dibuat
CREATE TABLE reservations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INTEGER,
    reservation_period TSRANGE,
    EXCLUDE USING gist (
        room_id WITH =,              -- ✅ Sekarang bisa!
        reservation_period WITH &&
    )
);
```

**Apa yang dilakukan btree_gist?**

- Menambahkan fungsionalitas B-tree (equality) ke GIST index
- Memungkinkan penggunaan operator `=` dalam GIST-based exclusion
- Menggabungkan kekuatan B-tree (equality) dan GIST (ranges)

**Catatan:**

- Extensions akan dibahas lebih detail di modul lain
- Untuk sekarang, cukup tahu bahwa btree_gist diperlukan untuk use case ini

### g. Partial EXCLUSION Constraint (dengan WHERE)

**Scenario real-world: Handling canceled reservations**

Problem dengan constraint sebelumnya:

```sql
CREATE TABLE reservations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INTEGER,
    reservation_period TSRANGE,
    booking_status TEXT,  -- 'confirmed', 'canceled', etc.
    EXCLUDE USING gist (
        room_id WITH =,
        reservation_period WITH &&
    )
);

-- Insert canceled reservation
INSERT INTO reservations
VALUES (DEFAULT, 1, '[2024-09-01, 2024-09-03)', 'canceled');
-- ✅ Berhasil

-- Coba book ulang dengan status confirmed (harusnya boleh!)
INSERT INTO reservations
VALUES (DEFAULT, 1, '[2024-09-01, 2024-09-03)', 'confirmed');
-- ❌ Error: Ditolak! (padahal yang pertama canceled)
```

**Solusi: Tambahkan predicate dengan WHERE clause**

```sql
CREATE TABLE reservations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INTEGER,
    reservation_period TSRANGE,
    booking_status TEXT,
    EXCLUDE USING gist (
        room_id WITH =,
        reservation_period WITH &&
    )
    WHERE (booking_status != 'canceled')  -- Predicate!
);
```

**Cara kerja:**

- Constraint **HANYA diterapkan** pada row dengan `booking_status != 'canceled'`
- Row dengan `booking_status = 'canceled'` **diabaikan** dari pengecekan
- Ini disebut **partial exclusion constraint**

**Contoh penggunaan:**

```sql
-- Insert canceled reservation
INSERT INTO reservations
VALUES (DEFAULT, 1, '[2024-09-01, 2024-09-03)', 'canceled');
-- ✅ Berhasil

-- Book ulang dengan confirmed (sekarang boleh!)
INSERT INTO reservations
VALUES (DEFAULT, 1, '[2024-09-01, 2024-09-03)', 'confirmed');
-- ✅ Berhasil! (canceled diabaikan dari constraint)

-- Coba insert confirmed kedua (conflict!)
INSERT INTO reservations
VALUES (DEFAULT, 1, '[2024-09-01, 2024-09-03)', 'confirmed');
-- ❌ Error: Tidak boleh 2 confirmed dengan room dan periode yang sama
```

**Visualisasi:**

```
Constraint hanya cek row dengan booking_status != 'canceled'

Row 1: canceled  → Diabaikan (tidak dicek)
Row 2: confirmed → Dicek ✅
Row 3: confirmed → Dicek dan conflict dengan Row 2 ❌
```

**Business logic:**

- Ketika reservasi di-cancel, slot menjadi available lagi
- Bisa di-book ulang dengan status berbeda
- Tapi tidak boleh ada 2+ reservasi aktif (non-canceled) yang conflict

### h. Rangkuman Sintaks EXCLUSION

**Template lengkap:**

```sql
CREATE TABLE table_name (
    column1 TYPE,
    column2 TYPE,
    ...
    EXCLUDE USING index_method (
        column1 WITH operator1,
        column2 WITH operator2,
        ...
    )
    WHERE (predicate)  -- Optional
);
```

**Komponen:**

- `USING gist`: Index method (biasanya gist untuk ranges)
- `column WITH operator`: Kolom dan operator untuk cek konflik
- `WHERE (predicate)`: Kondisi tambahan (partial constraint)

**Operator umum:**

- `=`: Equality (butuh btree_gist extension)
- `&&`: Overlap (untuk range types)
- `@>`: Contains
- `<@`: Contained by
- Dan operator lainnya tergantung tipe data

## 3. Hubungan Antar Konsep

### Alur Pemahaman:

```
1. UNIQUE Constraint (dari video sebelumnya)
   ↓
   Terbatas pada equality check (nilai harus exact berbeda)

2. EXCLUSION Constraint
   ↓
   Lebih fleksibel: bisa gunakan operator custom (tidak hanya =)

3. Simple EXCLUSION
   ↓
   Cek satu kolom saja (terlalu ketat)

4. Composite EXCLUSION
   ↓
   Cek kombinasi kolom (lebih realistic)
   ↓
   Butuh btree_gist untuk mix equality + ranges

5. Partial EXCLUSION
   ↓
   Tambahkan WHERE clause untuk business logic kompleks
```

### Diagram Comparison:

```
UNIQUE Constraint:
├─ Operator: = (equality only)
├─ Use case: Prevent exact duplicates
└─ Example: email, username

EXCLUSION Constraint:
├─ Operator: Custom (=, &&, @>, dll.)
├─ Use case: Prevent complex conflicts
└─ Example: overlapping reservations, spatial conflicts
```

### Evolusi Solusi Reservasi:

```
Problem 1: Overlapping reservations
└─ Solusi: EXCLUDE (reservation_period WITH &&)
   └─ Problem baru: Tidak bisa book room berbeda di waktu sama

Problem 2: Multi-room support
└─ Solusi: EXCLUDE (room_id WITH =, reservation_period WITH &&)
   └─ Butuh: btree_gist extension
   └─ Problem baru: Canceled reservation tidak bisa di-book ulang

Problem 3: Canceled reservations
└─ Solusi: Tambahkan WHERE (booking_status != 'canceled')
   └─ Final solution! ✅
```

### Dependency Tree:

```
EXCLUSION Constraint
├─ Requires: GIST index knowledge
├─ Requires: Range types (TSRANGE, DATERANGE)
├─ Requires: Operator knowledge (&&, =, @>)
├─ Optional: btree_gist extension (untuk equality)
└─ Optional: WHERE clause (untuk partial constraint)
```

## 4. Kesimpulan

**Key Takeaways:**

1. **EXCLUSION vs UNIQUE:**

   - UNIQUE: Prevent exact duplicates (operator `=` only)
   - EXCLUSION: Prevent custom conflicts (any operator)

2. **Struktur dasar:**

   ```sql
   EXCLUDE USING gist (column WITH operator)
   ```

3. **Composite exclusion:**

   - Combine multiple columns dengan operators berbeda
   - Example: `room_id WITH =, reservation_period WITH &&`

4. **btree_gist extension:**

   - Diperlukan untuk menggunakan operator `=` dalam GIST index
   - Enable dengan: `CREATE EXTENSION btree_gist`

5. **Partial exclusion dengan WHERE:**

   - Tambahkan predicate untuk business logic
   - Example: `WHERE (booking_status != 'canceled')`

6. **Use cases umum:**
   - Overlapping time ranges (reservasi, jadwal)
   - Spatial data conflicts
   - Resource allocation
   - Appointment scheduling

**Best Practices:**

- Gunakan EXCLUSION ketika UNIQUE tidak cukup
- Pertimbangkan apakah butuh composite exclusion
- Install btree_gist jika mix equality dengan ranges
- Gunakan partial exclusion untuk business rules kompleks
- EXCLUSION lebih jarang digunakan dari UNIQUE, tapi sangat powerful untuk kasus tertentu

**Pattern umum:**

```sql
-- Pattern untuk reservasi/booking system
EXCLUDE USING gist (
    resource_id WITH =,      -- Resource yang sama
    time_range WITH &&       -- Tidak boleh overlap
)
WHERE (status != 'canceled') -- Kecuali yang canceled
```

**Catatan penting:**

- Foreign key constraints akan dibahas di modul indexing
- GIST dan index types akan dijelaskan lebih detail nanti
- Extensions akan dibahas di modul terpisah
