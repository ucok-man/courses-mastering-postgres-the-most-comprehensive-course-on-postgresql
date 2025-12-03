# 📝 Bit String di PostgreSQL: Menyimpan Data Binary Secara Efisien

## 1. Ringkasan Singkat

Video ini membahas tipe data **bit string** di PostgreSQL, yaitu cara menyimpan data dalam bentuk **ones dan zeros (1 dan 0)**. Meskipun termasuk tipe data esoterik dan sulit dibaca oleh manusia, bit string sangat berguna untuk kasus tertentu seperti menyimpan banyak nilai boolean dalam satu kolom atau melakukan operasi bitwise. Video juga menjelaskan perbedaan antara `BIT` (fixed-length) dan `BIT VARYING` (variable-length), serta cara menggunakan **bit mask** untuk operasi filtering.

---

## 2. Konsep Utama

### a. Apa itu Bit String?

**Definisi:**
Bit string adalah string yang berisi **nilai on/off** yang direpresentasikan sebagai **1 (on)** dan **0 (off)**.

**Karakteristik:**

- Sangat **opaque** (tidak jelas): sulit bagi manusia untuk memahami arti setiap bit tanpa dokumentasi
- Cocok untuk situasi tertentu, tapi **tidak disarankan untuk digunakan di mana-mana**
- Bisa menjadi pengganti multiple boolean columns atau integer dengan bitwise operations

**Cara membuat bit string:**

```sql
-- Menggunakan prefix B
SELECT B'1000';
-- Output: 1000

-- Casting dari string biasa
SELECT '0001'::BIT(4);
-- Output: 0001

-- Cek tipe data
SELECT pg_typeof(B'1000');
-- Output: bit
```

**Perhatikan:**

- Prefix `B` menandakan bit literal
- `BIT(4)` artinya bit string dengan panjang **4 bit**

---

### b. Kapan Menggunakan Bit String?

**TIDAK disarankan untuk:**

- Menyimpan **2-3 nilai boolean** saja → gunakan kolom boolean terpisah lebih baik
- Situasi di mana **readability penting** → bit string sulit dibaca manusia

**DISARANKAN untuk:**

- Menyimpan **banyak nilai boolean** (misalnya 64+ flags) dalam satu kolom
- Menghindari pembuatan **puluhan kolom boolean** yang membuat tabel terlalu lebar
- Situasi di mana **bitwise operations** sering dilakukan

**Alternatif lain:**

- **Integer**: bisa digunakan untuk bitwise operations, tapi kurang eksplisit
- **JSON column**: lebih readable, tapi overhead lebih besar
- **Separate table**: lebih normalized, tapi perlu join

**Keuntungan bit string vs integer:**

```sql
-- Bit string: lebih jelas bahwa ini adalah 8 nilai dengan 2 yang aktif
B'00010001'

-- Integer: tidak jelas tanpa konversi mental
17
```

---

### c. Operasi Bit Mask

**Konsep bit mask:**
Bit mask adalah teknik untuk **menguji atau mengekstrak bit tertentu** dari bit string dengan cara membandingkannya dengan pola bit lain.

**Aturan operasi AND (`&`):**

- Jika bit **on (1) di kedua tempat** → hasil **1**
- Jika bit **on di salah satu tempat** → hasil **0**
- Jika bit **off (0) di kedua tempat** → hasil **0**

**Contoh kasus: Feature Flags**

Misalkan user memiliki feature flags:

- Feature 1: enabled
- Feature 2: disabled
- Feature 3: enabled

```sql
-- Feature flags user: feature 1 dan 3 aktif
SELECT B'101' AS user_features;
-- Output: 101

-- Mask untuk cek feature 1
SELECT B'101' & B'100' AS check_feature_1;
-- Output: 100 (berarti feature 1 aktif)

-- Mask untuk cek feature 2
SELECT B'101' & B'010' AS check_feature_2;
-- Output: 000 (berarti feature 2 tidak aktif)

-- Mask untuk cek feature 3
SELECT B'101' & B'001' AS check_feature_3;
-- Output: 001 (berarti feature 3 aktif)
```

**Diagram operasi AND:**

```
User Features:  1 0 1
Mask Feature 1: 1 0 0
              & -----
Result:         1 0 0  ← Feature 1 aktif!

User Features:  1 0 1
Mask Feature 2: 0 1 0
              & -----
Result:         0 0 0  ← Feature 2 tidak aktif
```

---

### d. Tipe Bit String: `BIT` vs `BIT VARYING`

PostgreSQL menyediakan dua tipe bit string:

**1. `BIT(n)` - Fixed Length**

- Panjang bit **harus tetap sesuai deklarasi**
- Jika insert dengan panjang berbeda → **error**

**2. `BIT VARYING(n)` - Variable Length**

- Panjang bit bisa **bervariasi** hingga batas maksimum
- Lebih fleksibel untuk data yang panjangnya tidak pasti

**Contoh implementasi:**

```sql
CREATE TABLE bits (
  bit_fixed BIT(3),           -- harus tepat 3 bit
  bit_var BIT VARYING(32)     -- bisa 1-32 bit
);

-- Insert yang VALID
INSERT INTO bits VALUES (
  B'001',                     -- tepat 3 bit ✓
  B'10101'                    -- 5 bit, masih di bawah 32 ✓
);

-- Insert yang ERROR
INSERT INTO bits VALUES (
  B'01',                      -- hanya 2 bit ✗
  B'001'
);
-- ERROR: bit string length 2 does not match type bit(3)

-- Insert yang VALID
INSERT INTO bits VALUES (
  B'001',                     -- tepat 3 bit ✓
  B'1'                        -- 1 bit, di bawah 32 ✓
);
```

---

### e. Use Case Real-World

**Contoh 1: User Permissions (8 permissions)**

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username TEXT,
  permissions BIT(8)  -- 8 jenis permission
);

-- Bit positions:
-- [0]: can_read
-- [1]: can_write
-- [2]: can_delete
-- [3]: can_admin
-- [4]: can_export
-- [5]: can_import
-- [6]: can_share
-- [7]: can_publish

INSERT INTO users (username, permissions)
VALUES ('admin', B'11111111');  -- semua permission

INSERT INTO users (username, permissions)
VALUES ('editor', B'11100000'); -- read, write, delete saja

-- Cek apakah user bisa delete (bit ke-2)
SELECT username,
       (permissions & B'00100000')::INT > 0 AS can_delete
FROM users;
```

**Contoh 2: Days of Week (7 days)**

```sql
CREATE TABLE events (
  id SERIAL PRIMARY KEY,
  name TEXT,
  active_days BIT(7)  -- Mon, Tue, Wed, Thu, Fri, Sat, Sun
);

INSERT INTO events (name, active_days)
VALUES ('Gym Class', B'1010100');  -- Mon, Wed, Fri

-- Cek apakah event aktif di hari Senin (bit pertama)
SELECT name,
       (active_days & B'1000000')::INT > 0 AS active_on_monday
FROM events;
```

---

## 3. Hubungan Antar Konsep

```
┌─────────────────────────────────────────────┐
│         Bit String Use Cases                │
└─────────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
    ┌───▼────┐             ┌────▼─────┐
    │  BIT   │             │BIT VARYING│
    │ (fixed)│             │(variable) │
    └───┬────┘             └────┬──────┘
        │                       │
        └───────────┬───────────┘
                    │
            ┌───────▼────────┐
            │  Bit Masking   │
            │   Operation    │
            └───────┬────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
    ┌───▼──────┐           ┌────▼─────┐
    │ Testing  │           │Extracting│
    │  Flags   │           │  Values  │
    └──────────┘           └──────────┘
```

**Alur pemikiran:**

1. **Identifikasi kebutuhan**: Apakah perlu menyimpan banyak boolean values?
2. **Pilih tipe**: `BIT` jika panjang tetap, `BIT VARYING` jika variabel
3. **Define bit positions**: Dokumentasikan arti setiap bit position
4. **Gunakan bit masking**: Untuk test atau extract nilai tertentu
5. **Consider alternatives**: JSON, separate table, atau boolean columns mungkin lebih baik tergantung context

**Trade-offs:**

| Aspek                | Bit String     | Boolean Columns | JSON          | Separate Table |
| -------------------- | -------------- | --------------- | ------------- | -------------- |
| **Storage**          | Sangat efisien | Moderate        | Moderate      | Moderate       |
| **Readability**      | Buruk          | Sangat baik     | Baik          | Sangat baik    |
| **Query complexity** | Tinggi         | Rendah          | Moderate      | Moderate       |
| **Flexibility**      | Rendah         | Tinggi          | Sangat tinggi | Sangat tinggi  |
| **Performance**      | Sangat cepat   | Cepat           | Moderate      | Perlu JOIN     |

---

## 4. Kesimpulan

Bit string adalah tipe data **specialized** di PostgreSQL untuk menyimpan data binary (1 dan 0). Meskipun sangat **efisien dalam storage** dan bagus untuk **bitwise operations**, bit string memiliki kelemahan utama yaitu **sulit dibaca manusia**.

**Gunakan bit string ketika:**

- Perlu menyimpan **banyak boolean flags** (64+)
- **Bitwise operations** sering dilakukan
- **Storage efficiency** sangat penting

**Hindari bit string ketika:**

- Hanya punya **2-3 boolean values** (gunakan kolom boolean saja)
- **Readability dan maintainability** adalah prioritas
- Tim tidak familiar dengan bitwise operations

**Key takeaways:**

- `BIT(n)`: fixed-length bit string
- `BIT VARYING(n)`: variable-length bit string
- Gunakan **bit masking** dengan operator `&` untuk test flags
- Selalu **dokumentasikan** arti setiap bit position
- **Consider alternatives**: tidak selalu bit string adalah solusi terbaik

Sebagai developer, Anda punya banyak pilihan (bit string, integer, boolean columns, JSON, separate table) — pilih yang paling sesuai dengan **kebutuhan dan context** project Anda.
