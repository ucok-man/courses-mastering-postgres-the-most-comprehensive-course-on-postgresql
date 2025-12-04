# 📝 Composite Types di PostgreSQL: Menggabungkan Multiple Types dalam Satu Unit

## 1. Ringkasan Singkat

Video ini membahas **composite types** di PostgreSQL, yaitu tipe data yang menggabungkan beberapa tipe data PostgreSQL menjadi **satu unit yang tidak terpisahkan**. Meskipun fitur ini menarik dan powerful, composite types **jarang digunakan** dalam praktik sehari-hari. Pembahasan mencakup cara membuat composite type, cara instantiate-nya, cara menyimpannya di tabel, dan cara mengakses field individual dari composite type. Video juga menjelaskan kapan sebaiknya menggunakan composite types dan alternatif apa yang lebih umum dipakai.

---

## 2. Konsep Utama

### a. Apa itu Composite Types?

#### Definisi

Composite type adalah tipe data kustom di PostgreSQL yang memungkinkan Anda **menggabungkan beberapa tipe data berbeda** menjadi **satu kesatuan**. Dengan kata lain, Anda bisa membuat sebuah “paket data” yang berisi banyak field, lalu memperlakukan paket tersebut sebagai satu tipe tunggal. Konsepnya mirip seperti `struct` di berbagai bahasa pemrograman.

#### Karakteristik Utama

Ada beberapa hal penting yang membuat composite type berbeda dengan tipe data lain:

- Ia membungkus beberapa kolom menjadi **satu unit informasi yang terkompresi** — bukan kolom terpisah.
- Secara konsep, ia sangat mirip dengan **struct**, di mana semua field berada dalam satu entitas terpadu.
- Karena merupakan satu kesatuan, composite type adalah **inseparable unit**, sehingga field-field di dalamnya tidak dapat diperlakukan sebagai entitas terisolasi.
- Composite type **tidak mendukung constraints** pada level definisi tipe. Artinya, Anda tidak bisa menambahkan aturan seperti `NOT NULL` atau `CHECK` langsung pada tipe tersebut.

#### Kapan Composite Types Berguna?

Meskipun jarang dipakai, composite types tetap memiliki fungsi di beberapa kasus tertentu:

- Saat Anda memiliki sekumpulan field yang **selalu muncul bersama** dan masuk akal untuk dijadikan satu paket.
- Banyak **extensions** PostgreSQL—seperti PostGIS—mengandalkan composite types untuk menyimpan data kompleks. Misalnya, sebuah `address` type dapat berisi hingga 18 field yang selalu digunakan secara bersamaan.
- Berguna untuk membuat **struktur data reusable** yang dapat digunakan di berbagai tabel tanpa memerlukan duplikasi definisi kolom.

Sebagai ilustrasi, berikut contoh sederhana pembuatan composite type:

```sql
CREATE TYPE full_name AS (
  first_name text,
  last_name text
);
```

Jika Anda menggunakan tipe ini dalam sebuah tabel:

```sql
CREATE TABLE users (
  id serial PRIMARY KEY,
  name full_name
);
```

Maka satu kolom `name` akan berisi dua nilai sekaligus (`first_name` dan `last_name`), dan keduanya hanya dapat diakses sebagai bagian dari satu unit data.

Untuk mengambil datanya:

```sql
SELECT
  (name).first_name,
  (name).last_name
FROM users;
```

Hasilnya menampilkan field yang ada di dalam composite type, tetapi secara penyimpanan tetap dianggap satu kolom.

#### Disclaimer Penting

Dalam praktik sehari-hari, composite types sebenarnya **jarang sekali digunakan**. Ada beberapa alasan mengapa alternatif berikut biasanya lebih disarankan:

- Menyimpan nilai sebagai **kolom terpisah** jauh lebih fleksibel.
- Menggunakan **JSON/JSONB** lebih cocok untuk data semi-terstruktur atau data yang skemanya dinamis.
- Membuat **tabel terpisah** dengan relasi (sesuai prinsip normalisasi database) sering kali lebih jelas dan lebih mudah dikelola.

Jadi, composite types tetap merupakan fitur kuat di PostgreSQL, tetapi penggunaannya banyak bergantung pada konteks dan kebutuhan khusus—dan sering kali, opsi lain lebih tepat.

### b. Cara Membuat Composite Type

#### Sintaks Dasar

Untuk membuat composite type, Anda mendefinisikan sebuah tipe baru yang berisi beberapa field dengan tipe masing-masing. Bentuk umumnya seperti berikut:

```sql
CREATE TYPE type_name AS (
  field1 type1,
  field2 type2,
  field3 type3
);
```

Hal yang perlu diperhatikan adalah penggunaan keyword `AS`. Bagian ini terlihat sederhana, tetapi sangat penting. Jika Anda menghilangkan `AS`, PostgreSQL akan menafsirkan perintah tersebut sebagai pembuatan tipe lain (misalnya enum atau domain), bukan composite type.

#### Contoh: Membuat Address Type

Anggap Anda ingin membuat tipe `address` yang berisi beberapa informasi alamat. Maka Anda bisa menuliskannya seperti ini:

```sql
CREATE TYPE address AS (
  number INT,
  street TEXT,
  city TEXT,
  state TEXT,
  zip TEXT
);
```

Mari kita uraikan:

- `address` adalah nama composite type yang Anda buat.
- Di dalamnya terdapat lima field: `number`, `street`, `city`, `state`, dan `zip`.
- Masing-masing field memiliki tipe data sendiri. Misalnya `number` adalah integer, sementara empat lainnya berupa text.
- Walaupun sintaksnya mirip definisi kolom dalam tabel, ingat bahwa **ini bukan tabel**. Composite type hanya mendefinisikan “bentuk data” yang nantinya bisa dipakai di kolom tabel lain.

#### Keterbatasan Composite Type

Composite type memiliki batasan penting, yaitu Anda **tidak bisa menambahkan constraints** langsung pada definisi tipe tersebut. Berikut contohnya:

```sql
-- ❌ Tidak diizinkan: menambahkan constraint pada field composite type
CREATE TYPE address AS (
  number INT NOT NULL,  -- ERROR!
  street TEXT
);
```

PostgreSQL akan menolak definisi ini karena composite type tidak mendukung constraints seperti `NOT NULL`, `CHECK`, atau aturan lainnya di level tipe.

Namun, Anda tetap bisa menambahkan constraint **di level tabel** yang menggunakan composite type tersebut. Misalnya:

```sql
-- ✅ Diperbolehkan: constraint diterapkan pada kolom tabel
CREATE TABLE addresses (
  id SERIAL PRIMARY KEY,
  addr address NOT NULL
);
```

Di sini, kolom `addr` bertipe `address`, dan Anda bebas memberi constraint seperti `NOT NULL` pada kolom tersebut. Cara ini membuat Anda tetap bisa mengontrol validasi data tanpa melanggar aturan composite type.

Dengan memahami sintaks, contoh, dan batasannya, Anda bisa menentukan apakah composite type memang cocok untuk kebutuhan struktur data Anda.

### c. Cara Instantiate (Membuat Instance) Composite Type

#### Metode 1: Menggunakan `ROW()` Constructor

Cara yang paling umum dan paling jelas untuk membuat instance dari composite type adalah menggunakan fungsi `ROW()`. Anda cukup mengisi nilai-nilai field sesuai urutan yang didefinisikan dalam composite type, lalu melakukan cast ke tipe yang Anda inginkan.

```sql
SELECT ROW(123, 'Main', 'Anytown', 'ST', '12345')::address;
-- Output: (123,Main,Anytown,ST,12345)
```

Jika Anda ingin memastikan bahwa hasilnya benar-benar bertipe `address`, Anda bisa melakukan pengecekan menggunakan `pg_typeof`:

```sql
SELECT pg_typeof(
  ROW(123, 'Main', 'Anytown', 'ST', '12345')::address
);
-- Output: address
```

Dengan metode ini, PostgreSQL akan mengenali bahwa Anda sedang membuat sebuah baris (row) yang kemudian dikonversi menjadi sebuah instance dari composite type `address`.

#### Metode 2: Shorthand Tanpa Keyword `ROW`

PostgreSQL juga menyediakan sintaks yang lebih ringkas. Anda dapat menulis nilai-nilai field langsung di dalam tanda kurung tanpa menuliskan keyword `ROW`. Namun, tanda kurung tetap wajib karena PostgreSQL perlu tahu bahwa Anda sedang membuat satu unit data.

```sql
SELECT (123, 'Main', 'Anytown', 'ST', '12345')::address;
-- Output: (123,Main,Anytown,ST,12345)
```

Secara fungsional, cara ini identik dengan penggunaan `ROW()`. Bedanya hanya di gaya penulisan yang lebih pendek.

#### Metode 3: String Literal (Tidak Disarankan)

Anda juga bisa membuat instance composite type dengan menuliskan seluruh nilainya sebagai string literal. Tetapi cara ini memiliki banyak kelemahan, terutama pada data teks yang membutuhkan penanganan tanda kutip.

```sql
SELECT '(123,"Main St","Any Town","ST","12345")'::address;
```

Mengapa metode ini tidak direkomendasikan?

- Anda harus menggunakan **double quoting** pada masing-masing field bertipe teks.
- Memunculkan banyak **nested quotes**, yang membuat kode lebih sulit dibaca dan rentan kesalahan.
- Secara keseluruhan, cara ini mudah menimbulkan bug, terutama ketika nilai field mengandung karakter khusus.

Karena alasan tersebut, cara terbaik dan paling aman adalah menggunakan `ROW()` atau shorthand parentheses, karena keduanya lebih konsisten, terbaca dengan jelas, dan minim error.

### d. Menyimpan Composite Type di Table

#### Membuat Tabel yang Memiliki Kolom Composite Type

Untuk menyimpan composite type ke dalam tabel, Anda cukup mendefinisikan sebuah kolom yang menggunakan tipe tersebut. Contohnya seperti berikut:

```sql
CREATE TABLE addresses (
  id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  addr address  -- kolom dengan tipe 'address'
);
```

Mari kita uraikan:

- Kolom `id` menggunakan sintaks modern `GENERATED ALWAYS AS IDENTITY`, yang artinya PostgreSQL akan mengisi nilai ini secara otomatis seperti auto-increment.
- Kolom `addr` bertipe `address`, yaitu composite type yang sudah Anda definisikan sebelumnya.
- Satu kolom `addr` sebenarnya menyimpan **lima field** sekaligus (`number`, `street`, `city`, `state`, `zip`). Semuanya dikemas sebagai satu unit data.

#### Menyisipkan Data ke Tabel

Untuk memasukkan nilai ke dalam kolom composite type, Anda bisa menggunakan `ROW()` ataupun shorthand parentheses.

Menggunakan `ROW()`:

```sql
INSERT INTO addresses (addr)
VALUES (
  ROW(123, 'Main', 'Anytown', 'ST', '12345')::address
);
```

Atau shorthand tanpa `ROW`:

```sql
INSERT INTO addresses (addr)
VALUES (
  (456, 'Elm St', 'Springfield', 'IL', '62701')::address
);
```

Kedua bentuk ini menghasilkan instance `address` yang valid.

#### Melihat Data yang Sudah Disimpan

Untuk melihat data di tabel:

```sql
SELECT * FROM addresses;
```

Output yang ditampilkan PostgreSQL akan mirip seperti ini:

| id  | addr                                |
| --- | ----------------------------------- |
| 1   | (123,Main,Anytown,ST,12345)         |
| 2   | (456,"Elm St",Springfield,IL,62701) |

Hal yang perlu diperhatikan:

- Nilai composite type ditampilkan sebagai **satu unit** di dalam tanda kurung.
- Field bertipe text yang memiliki spasi akan otomatis diberi tanda kutip oleh PostgreSQL.

Dengan pola ini, Anda dapat menyimpan struktur data yang kompleks dalam sebuah kolom, namun tetap mengaksesnya sebagai satu kesatuan.

### e. Mengakses Individual Fields dari Composite Type

#### Masalah: Sintaks yang Sedikit Menjebak

Saat pertama kali mencoba mengakses field di dalam composite type, banyak orang menulisnya seperti mengakses kolom dalam tabel:

```sql
SELECT addr.number FROM addresses;
```

Namun, ini akan menghasilkan error:

```
ERROR: missing FROM-clause entry for table "addr"
```

Mengapa demikian? Ketika PostgreSQL melihat `addr.number`, ia menganggap `addr` adalah **alias tabel**, bukan kolom yang berisi composite type. Karena tidak ada tabel bernama `addr`, PostgreSQL pun mengembalikan error.

#### Solusi: Bungkus Kolom dalam Parentheses

Untuk memberi tahu PostgreSQL bahwa `addr` adalah sebuah kolom bertipe composite, bukan alias tabel, Anda perlu membungkusnya dengan tanda kurung:

```sql
SELECT (addr).number FROM addresses;
-- Output: 123

SELECT (addr).street FROM addresses;
-- Output: Main

SELECT (addr).city FROM addresses;
-- Output: Anytown
```

Dengan `(addr).field`, PostgreSQL memahami bahwa Anda sedang mengakses field dari composite type.

#### Mengakses Banyak Field Sekaligus

Anda bisa mengambil beberapa field sekaligus seperti berikut:

```sql
SELECT
  id,
  (addr).number,
  (addr).street,
  (addr).city,
  (addr).state,
  (addr).zip
FROM addresses;
```

Contoh outputnya:

| id  | number | street | city        | state | zip   |
| --- | ------ | ------ | ----------- | ----- | ----- |
| 1   | 123    | Main   | Anytown     | ST    | 12345 |
| 2   | 456    | Elm St | Springfield | IL    | 62701 |

Di sini, setiap field diambil dari satu kolom composite type yang dikemas secara utuh.

#### Diagram Cara Kerjanya

```
addresses table
├── id: 1
└── addr: (123, Main, Anytown, ST, 12345)
            ↓
    Akses dengan (addr).field
            ↓
    ├── (addr).number  → 123
    ├── (addr).street  → Main
    ├── (addr).city    → Anytown
    ├── (addr).state   → ST
    └── (addr).zip     → 12345
```

Dengan pola ini, Anda bisa mengakses bagian mana pun dari composite type dengan sintaks yang konsisten dan mudah dibaca.

### f. Kapan Menggunakan Composite Types vs Alternatif

#### Situasi di Mana Composite Types Cocok Digunakan

Composite types tidak selalu menjadi pilihan utama dalam desain schema, tetapi ada beberapa kasus spesifik di mana pendekatan ini justru sangat efektif.

**1. Data yang Dihasilkan oleh Extension**

Beberapa extension PostgreSQL, seperti PostGIS, memang dirancang untuk mengembalikan composite type.
Contohnya:

```sql
SELECT ST_GeomFromText('POINT(0 0)');
```

Fungsi ini mengembalikan struktur kompleks yang terdiri dari berbagai komponen (misalnya x, y, dan SRID). Menggunakan composite type di sini adalah hal yang natural karena kita menerima satu paket data teknis yang tidak bisa dipisahkan.

**2. Kelompok Data yang Benar-Benar Tidak Bisa Dipisahkan**

Contoh yang baik adalah bilangan kompleks—real part dan imaginary part selalu berjalan bersama.

```sql
CREATE TYPE complex AS (
  real NUMERIC,
  imag NUMERIC
);
```

Dua nilai ini tidak pernah diperlakukan terpisah, sehingga membungkusnya dalam composite type membuat struktur data lebih jelas dan terorganisir.

**3. Mengembalikan Beberapa Nilai dari Sebuah Function**

Composite types juga sangat berguna untuk fungsi yang mengembalikan beberapa nilai sekaligus tanpa perlu membuat table khusus.

```sql
CREATE FUNCTION get_full_address(user_id INT)
RETURNS address AS $$
  -- logika fungsi
$$ LANGUAGE sql;
```

Dengan cara ini, Anda bisa mengirimkan satu unit informasi lengkap sebagai return value.

#### Alternatif yang Lebih Umum Digunakan

Dalam praktik sehari-hari, biasanya ada pilihan yang lebih fleksibel, lebih mudah di-query, atau lebih siap untuk dioptimalkan.

#### 1. Discrete Columns (Paling Umum dan Paling Mudah)

Ini adalah pendekatan paling standar.

```sql
CREATE TABLE addresses (
  id SERIAL PRIMARY KEY,
  number INT,
  street TEXT,
  city TEXT,
  state TEXT,
  zip TEXT
);
```

Contoh query sederhana:

```sql
SELECT number, street FROM addresses WHERE city = 'Anytown';
```

**Mengapa ini populer?**

- Struktur tabel lebih jelas dan eksplisit.
- Anda bisa melakukan indexing per kolom.
- Dapat memberikan constraint per kolom (NOT NULL, UNIQUE, CHECK).
- Query jauh lebih mudah karena tidak perlu syntax `(addr).field`.

#### 2. JSON/JSONB Column untuk Data Semi-Terstruktur

Jika schema berpotensi berubah atau tidak selalu konsisten, JSON/JSONB adalah pilihan fleksibel.

```sql
CREATE TABLE addresses (
  id SERIAL PRIMARY KEY,
  addr JSONB
);
```

Insert data:

```sql
INSERT INTO addresses (addr)
VALUES ('{"number": 123, "street": "Main", "city": "Anytown"}');
```

Query menggunakan operator JSON:

```sql
SELECT addr->>'street' FROM addresses;
```

**Keuntungan utamanya:**

- Mudah menambah atau menghapus field tanpa `ALTER TABLE`.
- Mendukung indexing menggunakan GIN.
- Ideal untuk data yang tidak selalu memiliki struktur tetap.

#### 3. Separate Table untuk Normalisasi

Jika satu entitas butuh menyimpan banyak item terkait (misalnya satu user memiliki banyak alamat), pendekatan tabel terpisah adalah pilihan terbaik.

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name TEXT
);

CREATE TABLE addresses (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id),
  number INT,
  street TEXT,
  city TEXT
);
```

**Keunggulan desain ini:**

- Struktur data lebih rapi dan sesuai prinsip normalisasi.
- Setiap entitas bisa di-query secara mandiri.
- Fleksibel untuk relasi satu-ke-banyak atau bahkan banyak-ke-banyak.

#### Tabel Perbandingan

Berikut ringkasan komparasi setiap pendekatan:

| Approach             | Readability | Flexibility | Query Ease | Use Case                         |
| -------------------- | ----------- | ----------- | ---------- | -------------------------------- |
| **Composite Type**   | Medium      | Low         | Medium     | Extension data, function returns |
| **Discrete Columns** | High        | Low         | High       | Fixed schema, frequent queries   |
| **JSON/JSONB**       | Medium      | Very High   | Medium     | Semi-structured, changing schema |
| **Separate Table**   | High        | High        | High       | Normalized data, one-to-many     |

Dengan memahami karakteristik masing-masing opsi, Anda dapat memilih desain yang paling tepat untuk kasus penggunaan tertentu—apakah ingin struktur data yang tidak terpisahkan, fleksibilitas tinggi, atau kemudahan dalam melakukan query dan indexing.

### g. Real-World Use Case: PostGIS Extension

**PostGIS menggunakan composite types secara ekstensif:**

```sql
-- PostGIS geocoder returns address composite type
SELECT
  (g.addy).address,
  (g.addy).streetname,
  (g.addy).streettypeabbrev,
  (g.addy).city,
  (g.addy).state,
  (g.addy).zip
FROM
  geocode('1600 Pennsylvania Ave, Washington DC') AS g;
```

**Composite type `addy` dari PostGIS:**

- Memiliki **18+ fields** untuk komponen address
- Returned oleh geocoding functions
- User perlu tahu cara access fields dengan `(g.addy).field` syntax

**Ini adalah use case yang bagus untuk composite types karena:**

- Data structure **sudah defined** oleh extension
- User **tidak perlu modify** structure
- **Function return value** yang complex
- Semua fields **semantically related**

---

## 3. Hubungan Antar Konsep

```
┌─────────────────────────────────────────────┐
│      COMPOSITE TYPE DEFINITION              │
│   CREATE TYPE name AS (fields...)           │
└─────────────────┬───────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼────────┐    ┌─────▼──────────┐
│  INSTANTIATE   │    │  USE IN TABLE  │
│  ROW(...):type │    │  col type_name │
└───────┬────────┘    └─────┬──────────┘
        │                   │
        └─────────┬─────────┘
                  │
        ┌─────────▼──────────┐
        │   STORE IN TABLE   │
        │  INSERT VALUES(ROW)│
        └─────────┬──────────┘
                  │
        ┌─────────▼──────────┐
        │   QUERY DATA       │
        ├────────────────────┤
        │ • SELECT (col).fld │
        │ • WHERE conditions │
        └────────────────────┘
```

**Alur kerja composite types:**

1. **Define composite type** dengan `CREATE TYPE ... AS (...)`

   - Tentukan semua fields dan types
   - Seperti membuat "blueprint" data structure

2. **Use in table definition** sebagai column type

   - Kolom akan menyimpan entire composite value
   - Treated sebagai single unit

3. **Insert data** menggunakan `ROW()` atau parentheses

   - Cast ke composite type dengan `::`
   - Semua fields harus provided (atau NULL)

4. **Query data** dengan syntax khusus
   - Wrap column dalam `(...)` untuk access fields
   - Syntax: `(column_name).field_name`

**Trade-offs decision tree:**

```
Perlu store group of related fields?
    │
    ├─ Yes → Fields ALWAYS appear together?
    │        │
    │        ├─ Yes → Data from extension?
    │        │        │
    │        │        ├─ Yes → Use COMPOSITE TYPE ✓
    │        │        │
    │        │        └─ No → Schema fixed forever?
    │        │                │
    │        │                ├─ Yes → Use DISCRETE COLUMNS ✓
    │        │                │
    │        │                └─ No → Use JSONB ✓
    │        │
    │        └─ No → Use SEPARATE TABLE ✓
    │
    └─ No → Use DISCRETE COLUMNS ✓
```

---

## 4. Kesimpulan

Composite types adalah fitur PostgreSQL untuk menggabungkan multiple types menjadi **single discrete unit**, namun **jarang digunakan** dalam praktik sehari-hari.

**Key takeaways:**

**Syntax essentials:**

- **Define**: `CREATE TYPE name AS (fields...)`
- **Instantiate**: `ROW(values)::typename` atau `(values)::typename`
- **Access fields**: `(column_name).field_name` (parentheses wajib!)

**Kapan menggunakan:**

- ✅ Data dari **extensions** (seperti PostGIS)
- ✅ **Function return types** yang complex
- ✅ Truly **inseparable data groups**

**Kapan TIDAK menggunakan (hampir selalu):**

- ❌ General application data → gunakan **discrete columns**
- ❌ Semi-structured data → gunakan **JSONB**
- ❌ One-to-many relationships → gunakan **separate table**

**Limitations:**

- Tidak bisa define **constraints** di type level
- Syntax **kurang intuitif** untuk access fields
- Sulit **modify structure** setelah dibuat
- Tidak bisa **partial update** individual fields easily

**Best practice:**

> "Composite types are cool, but you probably don't need them. Default to discrete columns or JSONB unless you have a very specific reason."

Composite types adalah tool yang powerful untuk **niche use cases**, terutama saat bekerja dengan extensions atau complex function returns. Namun untuk mayoritas aplikasi, **discrete columns atau JSONB** adalah pilihan yang lebih praktis dan maintainable.
