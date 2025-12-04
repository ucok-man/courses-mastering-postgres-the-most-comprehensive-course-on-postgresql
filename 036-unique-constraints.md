# 📝 UNIQUE Constraint di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang constraint UNIQUE di PostgreSQL, yang memastikan tidak ada nilai duplikat dalam satu kolom atau kombinasi beberapa kolom. Materi mencakup perbedaan antara PRIMARY KEY dan UNIQUE, perilaku khusus UNIQUE dengan nilai NULL, serta berbagai cara mendeklarasikan constraint UNIQUE termasuk composite unique constraint.

## 2. Konsep Utama

### a. PRIMARY KEY vs UNIQUE

#### Memahami Hubungan PRIMARY KEY dan UNIQUE

Ketika kita membuat sebuah _PRIMARY KEY_ pada sebuah tabel, sebenarnya database tidak hanya menambahkan satu jenis aturan. Sebaliknya, sebuah _PRIMARY KEY_ secara otomatis merupakan kombinasi dari dua constraint sekaligus: `NOT NULL` dan `UNIQUE`. Artinya, kolom yang dijadikan _PRIMARY KEY_ tidak boleh berisi nilai `NULL`, dan setiap nilainya harus unik di seluruh tabel.

Perhatikan contoh berikut:

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
    -- PRIMARY KEY otomatis menambahkan:
    -- 1. NOT NULL constraint
    -- 2. UNIQUE constraint
);
```

Pada contoh di atas, kolom `id` akan selalu memiliki nilai (tidak boleh `NULL`) dan nilai tersebut harus unik untuk setiap baris. PostgreSQL juga akan membuat index untuk kolom ini agar pencarian berdasarkan `id` menjadi lebih cepat.

#### Mengapa PRIMARY KEY Hanya Satu?

Dalam satu tabel, kita biasanya hanya memiliki satu _PRIMARY KEY_. Alasannya adalah karena _PRIMARY KEY_ digunakan sebagai identitas utama setiap baris — layaknya nomor KTP pada manusia. Database membutuhkan satu identitas yang paling jelas untuk menjadi acuan referensi, terutama ketika tabel lain membuat hubungan melalui _foreign key_.

Namun, memiliki satu _PRIMARY KEY_ tidak berarti kita tidak bisa memiliki kolom lain yang juga harus unik.

#### UNIQUE untuk Kolom Tambahan

Jika ada kolom lain yang memang perlu memiliki nilai unik — misalnya `email` pada tabel `users` atau `sku` pada tabel `products` — kita bisa menambahkan constraint `UNIQUE`.

Contoh penerapannya:

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku TEXT UNIQUE,          -- setiap SKU harus unik, tetapi bisa NULL
    name TEXT NOT NULL
);
```

Di sini:

- `id` bertindak sebagai identitas utama.
- `sku` harus unik, tetapi berbeda dengan PRIMARY KEY, kolom ini masih boleh berisi `NULL` (kecuali ditambah constraint NOT NULL).
- Kita dapat memiliki banyak `UNIQUE constraint` untuk kolom yang berbeda-beda.

Dengan memahami perbedaan dan hubungan antara PRIMARY KEY dan UNIQUE, kita bisa merancang struktur tabel yang lebih teratur, konsisten, dan optimal untuk kebutuhan aplikasi.

### b. UNIQUE Constraint Dasar

#### Memahami Fungsi Dasar UNIQUE

Constraint `UNIQUE` digunakan ketika kita ingin memastikan bahwa sebuah kolom tidak memiliki nilai yang sama dengan baris lain. Dengan kata lain, setiap nilai dalam kolom tersebut harus benar-benar unik. Constraint ini sangat berguna untuk data yang memang tidak boleh duplikasi, seperti nomor produk, email pengguna, atau kode referensi tertentu.

Mari kita lihat contoh sederhana berikut:

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_number TEXT UNIQUE  -- Nilai harus unik
);
```

Pada tabel `products` di atas, kolom `product_number` diberikan constraint `UNIQUE`. Artinya, PostgreSQL akan menolak setiap operasi `INSERT` atau `UPDATE` yang mencoba memasukkan nilai yang sama dengan nilai yang sudah ada.

#### Contoh Perilaku INSERT

Perhatikan dua operasi berikut:

```sql
-- Insert pertama berhasil
INSERT INTO products VALUES (DEFAULT, 'ABC123');  -- ✅ Berhasil

-- Insert kedua dengan nilai sama akan gagal
INSERT INTO products VALUES (DEFAULT, 'ABC123');  -- ❌ Error: already exists
```

Pada `INSERT` pertama, PostgreSQL menerima nilai `'ABC123'` karena kolom `product_number` masih kosong dan tidak ada konflik nilai.

Namun, `INSERT` kedua mencoba memasukkan nilai yang sama. Karena constraint `UNIQUE` mengharuskan nilai tersebut tidak boleh duplikat, database langsung menolak operasi tersebut.

#### Pesan Error yang Muncul

Jika kita mencoba memasukkan nilai yang bertentangan dengan aturan `UNIQUE`, PostgreSQL memberikan pesan error yang cukup jelas:

```
ERROR: duplicate key value violates unique constraint
DETAIL: Key (product_number)=(ABC123) already exists.
```

Bagian penting dari pesan tersebut adalah detail yang menunjukkan kolom mana yang melanggar (`product_number`) dan nilai apa yang menyebabkan konflik (`ABC123`).

Dengan memahami konsep dasar ini, kita bisa mendesain struktur tabel yang lebih aman, terhindar dari data duplikat, dan mudah diandalkan untuk kebutuhan bisnis atau aplikasi.

### c. UNIQUE dan NULL - Perilaku Khusus

#### Memahami Cara UNIQUE Memperlakukan NULL

Salah satu hal yang sering membuat bingung ketika bekerja dengan constraint `UNIQUE` adalah bagaimana database memperlakukan nilai `NULL`. Secara default, PostgreSQL (dan sebagian besar SQL database lain) menganggap nilai `NULL` sebagai _distinct_, atau dianggap berbeda satu sama lain. Ini berarti walaupun sebuah kolom memiliki constraint `UNIQUE`, Anda tetap bisa memasukkan banyak baris dengan nilai `NULL` tanpa menimbulkan error.

Mari kita lihat contoh berikut:

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_number TEXT UNIQUE  -- Nullable by default
);

-- Semua insert ini BERHASIL karena NULL ≠ NULL
INSERT INTO products VALUES (DEFAULT, NULL);  -- ✅ Berhasil
INSERT INTO products VALUES (DEFAULT, NULL);  -- ✅ Berhasil juga!
INSERT INTO products VALUES (DEFAULT, NULL);  -- ✅ Masih berhasil!

SELECT * FROM products;
```

Jika kita menjalankan query tersebut, inilah hasilnya:

```
id | product_number
---|---------------
1  | NULL
2  | NULL
3  | NULL
```

Terlihat bahwa meskipun ada constraint `UNIQUE` pada kolom `product_number`, memasukkan beberapa nilai `NULL` tetap dianggap valid.

#### Mengapa Hal Ini Diizinkan?

Untuk memahami perilaku ini, kita perlu mengingat kembali konsep fundamental tentang `NULL`: **NULL mewakili nilai yang tidak diketahui (unknown value)**.

Dari konsep ini muncul beberapa implikasi penting:

- `NULL ≠ NULL`
  Logikanya, jika nilai tersebut _tidak diketahui_, maka database tidak bisa memastikan apakah dua `NULL` mewakili nilai yang sama atau berbeda.

- Karena ketidakpastian tersebut, database tidak memperlakukan dua `NULL` sebagai duplikat.

- Hasilnya: constraint `UNIQUE` tidak menganggap dua `NULL` sebagai pelanggaran, sehingga beberapa baris dengan nilai `NULL` tetap diperbolehkan.

Perilaku ini penting dipahami ketika merancang tabel. Jika Anda memiliki kolom yang harus benar-benar unik dan tidak boleh berisi banyak `NULL`, maka perlu menambahkan `NOT NULL` secara eksplisit. Tanpa itu, constraint `UNIQUE` saja tidak cukup untuk mencegah banyak nilai `NULL` masuk ke dalam tabel.

### d. Menangani NULL dalam UNIQUE Constraint

#### Memahami Pilihan Penanganan NULL

Seperti yang sudah dijelaskan sebelumnya, constraint `UNIQUE` secara default memperbolehkan banyak nilai `NULL` dalam satu kolom. Namun dalam beberapa kasus, Anda mungkin ingin mengontrol bagaimana `NULL` diperlakukan—apakah tidak boleh sama sekali, atau hanya boleh satu kali saja. PostgreSQL menyediakan tiga pendekatan berbeda untuk kebutuhan ini.

---

#### Opsi 1: Membuat Kolom NOT NULL

Opsi ini paling sederhana: jika Anda tidak ingin ada nilai `NULL` sama sekali, tambahkan constraint `NOT NULL` pada kolom yang memiliki `UNIQUE`.

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_number TEXT NOT NULL UNIQUE  -- Tidak boleh NULL
);

-- Ini akan error
INSERT INTO products VALUES (DEFAULT, NULL);  -- ❌ Error: NOT NULL violation
```

Dengan pendekatan ini:

- Semua baris wajib memiliki nilai untuk kolom `product_number`.
- Constraint `UNIQUE` bekerja sepenuhnya karena tidak ada `NULL` yang perlu ditangani secara khusus.
- Cocok untuk kolom seperti email, username, SKU, dan data lain yang harus selalu terisi.

---

#### Opsi 2: Menggunakan UNIQUE NULLS NOT DISTINCT

PostgreSQL menyediakan fitur tambahan bernama `NULLS NOT DISTINCT` untuk kasus ketika Anda masih mengizinkan nilai `NULL`, tetapi hanya satu kali saja.

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_number TEXT UNIQUE NULLS NOT DISTINCT
);

-- Insert NULL pertama berhasil
INSERT INTO products VALUES (DEFAULT, NULL);  -- ✅ Berhasil

-- Insert NULL kedua GAGAL
INSERT INTO products VALUES (DEFAULT, NULL);  -- ❌ Error: duplicate key
```

Perilaku ini berbeda dari `UNIQUE` biasa karena:

- `NULL` dianggap **setara** dengan `NULL` lainnya
  (berlawanan dengan perilaku standar SQL yang menganggap `NULL` serba tidak pasti).
- Karena dianggap sama, hanya **satu** `NULL` yang diperbolehkan dalam kolom tersebut.
- Cocok untuk situasi:
  “Boleh ada satu nilai yang masih unknown, tapi lebih dari itu tidak masuk akal.”

#### Karakteristik tambahan

- Bersifat non-standard SQL (fitur khusus PostgreSQL).
- Berguna ketika hanya satu nilai `NULL` yang dianggap valid untuk menjaga konsistensi data.

---

#### Diagram Perilaku untuk Mempermudah Pemahaman

```
Default UNIQUE:
NULL, NULL, NULL, 'A', 'B' → ✅ Valid (multiple NULLs allowed)

UNIQUE NULLS NOT DISTINCT:
NULL, 'A', 'B' → ✅ Valid (one NULL allowed)
NULL, NULL, 'A' → ❌ Invalid (duplicate NULL)

NOT NULL UNIQUE:
'A', 'B', 'C' → ✅ Valid (no NULLs at all)
NULL, 'A' → ❌ Invalid (NULL not allowed)
```

Dengan memahami ketiga opsi ini, Anda bisa memilih perilaku yang paling sesuai dengan kebutuhan bisnis dan desain database Anda—apakah ingin membiarkan banyak `NULL`, hanya satu `NULL`, atau tidak mengizinkan `NULL` sama sekali.

### e. Composite UNIQUE Constraint

#### Memahami UNIQUE pada Beberapa Kolom Sekaligus

Selain diterapkan pada satu kolom, constraint `UNIQUE` juga bisa diterapkan pada kombinasi beberapa kolom. Dalam situasi tertentu, sebuah kolom boleh saja memiliki nilai yang sama selama kombinasi dengan kolom lain tetap unik. Konsep ini disebut **composite unique constraint**.

Mari kita lihat contoh tabel `products` berikut:

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    brand TEXT NOT NULL,
    product_number TEXT NOT NULL,
    UNIQUE (brand, product_number)  -- Kombinasi harus unik
);
```

Dalam definisi tabel ini, baik `brand` maupun `product_number` diperbolehkan memiliki duplikasi secara individual. Namun, pasangan nilai `(brand, product_number)` harus unik. Artinya, tidak boleh ada dua baris yang memiliki kombinasi brand _dan_ product_number yang sama persis.

#### Contoh Insert dan Penjelasan

```sql
-- Insert dengan kombinasi berbeda
INSERT INTO products VALUES (DEFAULT, 'ABC', '123');  -- ✅ Berhasil
INSERT INTO products VALUES (DEFAULT, 'DEF', '123');  -- ✅ Berhasil (brand berbeda)
INSERT INTO products VALUES (DEFAULT, 'ABC', '456');  -- ✅ Berhasil (number berbeda)

-- Insert dengan kombinasi yang sama
INSERT INTO products VALUES (DEFAULT, 'ABC', '123');  -- ❌ Error: composite unique violation
```

Tiga insert pertama berhasil karena setiap kombinasi yang dimasukkan unik:

- `'ABC' + '123'`
- `'DEF' + '123'`
- `'ABC' + '456'`

Ketiganya berbeda satu sama lain. Namun, insert keempat gagal karena mencoba memasukkan kombinasi `'ABC' + '123'` yang sudah ada di baris pertama.

#### Melihat Hasilnya

Jika kita melihat isi tabel:

```sql
SELECT * FROM products;
```

Hasilnya:

```
id | brand | product_number
---|-------|---------------
1  | ABC   | 123
2  | DEF   | 123           ← 123 muncul 2x, tapi brand berbeda  → valid
3  | ABC   | 456           ← ABC muncul 2x, tapi number berbeda → valid
```

Dari hasil tersebut terlihat jelas bahwa:

- `product_number` dapat muncul lebih dari sekali, selama brand-nya berbeda.
- `brand` juga boleh muncul lebih dari sekali, selama product_number berbeda.
- Yang benar-benar harus unik hanyalah **pasangan** dari dua kolom tersebut.

#### Poin Penting

- Composite unique constraint memastikan keunikan berdasarkan kombinasi beberapa kolom.
- Kolom individual masih boleh memiliki nilai duplikat.
- Teknik ini sering digunakan pada relasi many-to-many, SKU per brand, atau data yang membutuhkan identifikasi berdasarkan dua atribut sekaligus.

Dengan composite UNIQUE, kita memiliki fleksibilitas lebih dalam merancang struktur data yang realistis tanpa mengorbankan integritas atau konsistensi.

### f. Deklarasi UNIQUE sebagai Table Constraint

#### Memahami Berbagai Cara Mendeklarasikan UNIQUE

Constraint `UNIQUE` bisa didefinisikan dengan beberapa gaya penulisan, dan masing-masing memiliki kegunaannya sendiri. Secara fungsi, semua cara ini menghasilkan perilaku yang sama: memastikan nilai di kolom (atau kombinasi kolom) tidak duplikat. Namun, dari sisi desain dan keterbacaan, pilihan gaya deklarasi dapat membuat struktur tabel lebih jelas dan mudah dirawat.

Berikut tiga cara umum untuk mendefinisikan `UNIQUE`.

#### Cara 1: Inline Column Constraint (langsung pada kolom)

Ini adalah cara yang paling ringkas dan sering digunakan untuk kolom tunggal.

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    product_number TEXT UNIQUE  -- Inline
);
```

Deklarasi ini ditulis tepat setelah tipe data kolom. Format ini cocok ketika constraint hanya berlaku pada satu kolom dan Anda ingin penulisan tetap singkat dan rapi.

#### Cara 2: Table Constraint (ditulis di bagian bawah definisi kolom)

Pendekatan ini menuliskan constraint sebagai bagian dari definisi tabel, bukan langsung pada kolomnya.

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    product_number TEXT,
    UNIQUE (product_number)  -- Di akhir sebagai table constraint
);
```

Meskipun fungsinya sama dengan inline constraint, gaya ini lebih fleksibel—terutama saat Anda perlu mendeklarasikan `UNIQUE` pada beberapa kolom sekaligus. Selain itu, penempatannya membuat constraint terlihat lebih eksplisit saat membaca struktur tabel.

#### Cara 3: Table Constraint dengan Nama (named constraint)

Dengan menambahkan nama pada constraint, Anda bisa memberikan konteks lebih jelas dan mendapatkan pesan error yang lebih informatif.

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    product_number TEXT,
    CONSTRAINT must_be_unique UNIQUE (product_number)  -- Dengan nama custom
);
```

Jika terjadi pelanggaran constraint, PostgreSQL akan menampilkan nama constraint tersebut:

```
ERROR: duplicate key value violates unique constraint "must_be_unique"
```

Pendekatan ini sangat berguna ketika Anda bekerja pada sistem besar, atau saat debugging di mana pesan error yang jelas dapat menghemat banyak waktu.

#### Kapan Menggunakan Masing-Masing Cara?

| Cara             | Kapan Digunakan                                                                                             |
| ---------------- | ----------------------------------------------------------------------------------------------------------- |
| Inline           | Saat mengatur `UNIQUE` untuk satu kolom saja dan ingin penulisan sederhana.                                 |
| Table constraint | Saat membuat composite unique atau ingin struktur constraint terlihat lebih konsisten.                      |
| Named constraint | Saat Anda menginginkan pesan error yang lebih jelas atau ingin mengorganisir constraint dengan nama khusus. |

Memahami variasi deklarasi ini membantu Anda menulis definisi tabel yang lebih bersih, konsisten, dan mudah dipelihara, terutama ketika sistem tumbuh dan struktur database menjadi semakin kompleks.

### g. Ringkasan Opsi UNIQUE

#### Tinjauan Lengkap Semua Variasi UNIQUE

Bagian ini merangkum seluruh opsi yang dapat Anda gunakan ketika bekerja dengan constraint `UNIQUE` di PostgreSQL. Setiap opsi memiliki karakteristik dan kegunaan yang berbeda—mulai dari mengizinkan banyak `NULL`, melarang `NULL` sama sekali, hingga membuat kombinasi kolom yang harus unik. Dengan memahami seluruh variasi ini, Anda dapat memilih pendekatan yang paling sesuai untuk kebutuhan desain tabel Anda.

```sql
-- Opsi 1: Basic unique (multiple NULLs allowed)
product_number TEXT UNIQUE
```

Opsi paling dasar. PostgreSQL memperbolehkan lebih dari satu `NULL`, karena secara default `NULL` dianggap memiliki nilai yang tidak bisa dibandingkan sehingga tidak dianggap duplikat.

```sql
-- Opsi 2: No NULLs at all
product_number TEXT NOT NULL UNIQUE
```

Kombinasi `NOT NULL` dan `UNIQUE`. Dengan ini, kolom tidak boleh kosong dan nilai harus benar-benar unik. Cocok untuk kolom yang wajib terisi seperti email, username, atau SKU.

```sql
-- Opsi 3: Only one NULL allowed
product_number TEXT UNIQUE NULLS NOT DISTINCT
```

Perilaku spesial: hanya **satu** `NULL` yang diizinkan. `NULL` diperlakukan seolah-olah memiliki nilai yang bisa dibandingkan dengan `NULL` lainnya. Jika ada `NULL` kedua, maka dianggap pelanggaran duplikasi.

```sql
-- Opsi 4: Composite unique
UNIQUE (brand, product_number)
```

UNIQUE untuk kombinasi kolom. Kolom individu boleh duplikat, tetapi kombinasinya harus unik. Sangat berguna saat dua atribut bersama-sama menjadi identitas unik suatu data, misalnya SKU per brand.

```sql
-- Opsi 5: Table constraint
UNIQUE (product_number)
```

Versi table-level dari opsi dasar. Fungsinya sama seperti inline, tetapi penulisannya berada di bagian bawah definisi tabel. Lebih fleksibel ketika Anda ingin konsistensi penulisan atau menyiapkan struktur composite.

```sql
-- Opsi 6: Named constraint
CONSTRAINT constraint_name UNIQUE (product_number)
```

Memberikan nama khusus pada constraint. Keuntungannya adalah error message menjadi lebih jelas karena menyebutkan nama constraint tersebut. Ini bermanfaat dalam debugging atau project besar.

```sql
-- Opsi 7: Composite dengan nulls not distinct
UNIQUE NULLS NOT DISTINCT (brand, product_number)
```

Gabungan antara composite unique dan perilaku “hanya satu NULL” pada masing-masing kolom yang masuk dalam kombinasi. Kombinasi `(brand, product_number)` harus unik dengan aturan bahwa nilai NULL di kolom tersebut tidak boleh muncul lebih dari satu.

Semua variasi ini memberi Anda fleksibilitas penuh untuk mengelola keunikan data sesuai kebutuhan bisnis dan struktur aplikasi Anda.

## 3. Hubungan Antar Konsep

### Alur Pemahaman:

```
1. PRIMARY KEY
   ↓
   Terdiri dari: NOT NULL + UNIQUE

2. UNIQUE Constraint
   ↓
   Bisa diterapkan pada kolom lain selain primary key

3. UNIQUE dengan NULL
   ↓
   Default: Multiple NULLs allowed (karena NULL ≠ NULL)

4. Solusi untuk NULL
   ├─ NOT NULL UNIQUE → No NULLs allowed
   └─ NULLS NOT DISTINCT → One NULL allowed

5. Composite UNIQUE
   ↓
   Kombinasi kolom harus unik (bukan individual)

6. Deklarasi
   ├─ Inline column constraint
   ├─ Table constraint
   └─ Named constraint
```

### Diagram Decision Tree:

```
Apakah butuh constraint UNIQUE?
├─ Pada PRIMARY KEY?
│  └─ Tidak perlu deklarasi UNIQUE (sudah otomatis)
│
└─ Pada kolom lain?
   ├─ Single column?
   │  ├─ Boleh NULL?
   │  │  ├─ Ya, multiple NULLs OK → TEXT UNIQUE
   │  │  ├─ Ya, tapi max 1 NULL → TEXT UNIQUE NULLS NOT DISTINCT
   │  │  └─ Tidak boleh NULL → TEXT NOT NULL UNIQUE
   │  │
   │  └─ Inline atau table constraint?
   │     ├─ Inline → TEXT UNIQUE
   │     └─ Table → UNIQUE (column_name)
   │
   └─ Multiple columns?
      └─ UNIQUE (col1, col2, ...)
```

### Hubungan dengan Konsep NULL:

Pemahaman UNIQUE constraint sangat bergantung pada pemahaman NULL dari video sebelumnya:

- **NULL = unknown value**
- **NULL ≠ NULL** dalam logika database
- Ini menjelaskan mengapa default UNIQUE mengizinkan multiple NULLs
- `NULLS NOT DISTINCT` adalah override dari perilaku default ini

## 4. Kesimpulan

**Key Takeaways:**

1. **PRIMARY KEY = NOT NULL + UNIQUE** secara otomatis
2. **UNIQUE constraint bisa ditambahkan pada kolom atau kombinasi kolom mana pun**
3. **Perilaku default UNIQUE dengan NULL:**
   - Multiple NULLs diizinkan (karena NULL ≠ NULL)
   - Ini mungkin atau mungkin tidak sesuai kebutuhan bisnis
4. **Tiga strategi menangani NULL:**
   - `NOT NULL UNIQUE`: Tidak boleh NULL sama sekali
   - `UNIQUE`: Multiple NULLs allowed (default)
   - `UNIQUE NULLS NOT DISTINCT`: Maksimal satu NULL
5. **Composite UNIQUE:** Kombinasi kolom harus unik, bukan kolom individual
6. **Cara deklarasi:** Inline, table constraint, atau named constraint - semuanya valid

**Best Practice:**

- Pertimbangkan apakah kolom boleh NULL atau tidak terlebih dahulu
- Untuk sebagian besar kasus, `NOT NULL UNIQUE` adalah pilihan paling jelas
- Gunakan `NULLS NOT DISTINCT` hanya jika ada use case spesifik untuk "satu nilai unknown"
- Gunakan composite unique untuk business rules yang melibatkan kombinasi kolom
- Beri nama constraint jika ingin error message lebih deskriptif
