# 📝 Bit String di PostgreSQL: Menyimpan Data Binary Secara Efisien

## 1. Ringkasan Singkat

Video ini membahas tipe data **bit string** di PostgreSQL, yaitu cara menyimpan data dalam bentuk **ones dan zeros (1 dan 0)**. Meskipun termasuk tipe data esoterik dan sulit dibaca oleh manusia, bit string sangat berguna untuk kasus tertentu seperti menyimpan banyak nilai boolean dalam satu kolom atau melakukan operasi bitwise. Video juga menjelaskan perbedaan antara `BIT` (fixed-length) dan `BIT VARYING` (variable-length), serta cara menggunakan **bit mask** untuk operasi filtering.

---

## 2. Konsep Utama

### a. Apa itu Bit String?

#### Definisi

Bit string adalah sebuah string yang berisi urutan nilai **on/off**, di mana setiap nilai direpresentasikan sebagai **1 untuk on** dan **0 untuk off**. Representasi ini sangat sederhana, tetapi tetap kuat untuk melakukan operasi yang berhubungan dengan bit atau flag.

#### Karakteristik

Bit string memiliki sifat yang cenderung **opaque**, artinya maknanya tidak langsung terlihat. Tanpa dokumentasi atau penjelasan, sulit untuk memahami arti dari setiap bit. Karena itu, meskipun efektif untuk kasus tertentu, tipe data ini **tidak disarankan dipakai secara sembarangan**, terutama jika tujuan utamanya adalah keterbacaan.
Dalam praktiknya, bit string bisa digunakan sebagai:

- Pengganti beberapa kolom boolean yang menangani flag.
- Mekanisme penyimpanan nilai status dalam bentuk integer yang kemudian diolah dengan operasi bitwise.

#### Cara Membuat Bit String

PostgreSQL menyediakan beberapa cara untuk membentuk bit string, dan berikut adalah contoh yang paling umum.

```sql
-- Menggunakan prefix B untuk bit literal
SELECT B'1000';
-- Output: 1000

-- Melakukan casting dari string biasa
SELECT '0001'::BIT(4);
-- Output: 0001

-- Mengecek tipe datanya
SELECT pg_typeof(B'1000');
-- Output: bit
```

Pada contoh pertama, prefix `B` menunjukkan bahwa nilai tersebut adalah **bit literal**, bukan string biasa.
Pada contoh kedua, `'0001'::BIT(4)` berarti Anda melakukan casting string `'0001'` ke tipe **BIT** dengan panjang **4 bit**. Jika jumlah karakter tidak sesuai dengan panjang bit yang ditentukan, PostgreSQL akan menyesuaikannya atau mengembalikan error tergantung kasusnya.

#### Perhatikan

Beberapa poin penting yang perlu diingat:

- Prefix **`B`** selalu menandakan bit literal.
- Penulisan **`BIT(4)`** berarti tipe tersebut hanya dapat menyimpan tepat **4 bit**. Jika ingin menyimpan bit string dengan panjang bervariasi, ada juga tipe `VARBIT` yang sifatnya lebih fleksibel.

Dengan memahami dasar-dasar ini, Anda dapat menentukan kapan bit string bermanfaat dan kapan sebaiknya memakai tipe data yang lebih deskriptif seperti boolean atau enum.

### b. Kapan Menggunakan Bit String?

#### Situasi yang Tidak Disarankan

Bit string bukan pilihan yang ideal ketika Anda hanya perlu menyimpan **sedikit nilai boolean**, misalnya dua atau tiga flag saja. Dalam kasus seperti ini, menggunakan kolom boolean terpisah jauh lebih jelas dan mudah dibaca.
Selain itu, jika konteks aplikasi Anda menuntut **keterbacaan yang tinggi**, bit string sebaiknya dihindari karena manusia sulit memahami arti setiap bit tanpa dokumentasi tambahan.

#### Situasi yang Disarankan

Bit string menjadi sangat berguna ketika Anda perlu menyimpan **banyak nilai boolean** dalam satu kolom—misalnya puluhan atau bahkan lebih dari 64 flag. Dengan cara ini, Anda tidak perlu membuat tabel dengan puluhan kolom boolean yang justru membuat struktur menjadi lebar dan sulit dikelola.
Selain itu, bit string juga cocok jika Anda sering melakukan **bitwise operations**, seperti mengecek flag tertentu, menggabungkan flag, atau memodifikasinya. PostgreSQL menyediakan operasi bitwise yang efisien sehingga penggunaan bit string dapat memberikan keuntungan performa.

#### Alternatif Lain

Meskipun bit string berguna, ada beberapa alternatif yang bisa dipertimbangkan tergantung kebutuhan:

- **Integer**: mendukung operasi bitwise dan efisien, tetapi kurang eksplisit. Ketika melihat nilai seperti `17`, kita tidak langsung tahu flag mana yang aktif.
- **Kolom JSON**: lebih mudah dibaca manusia, dan struktur datanya fleksibel. Namun, JSON memiliki overhead dalam hal penyimpanan dan performa.
- **Tabel terpisah**: paling sesuai secara normalisasi karena setiap flag bisa direpresentasikan sebagai baris. Namun, metode ini memerlukan join tambahan yang kadang menambah kompleksitas.

#### Keuntungan Bit String Dibandingkan Integer

Contoh berikut menunjukkan mengapa bit string bisa lebih informatif daripada integer dalam konteks representasi flag.

```sql
-- Bit string: kita bisa melihat langsung bahwa ada 8 bit,
-- dan bit ke-4 serta bit terakhir aktif.
B'00010001'

-- Integer: angkanya hanya 17,
-- tetapi untuk tahu bit mana yang aktif, kita harus menghitung atau mengonversinya.
17
```

Dengan bit string, struktur dan posisi bit yang aktif terlihat jelas sejak awal. Ini membuatnya lebih mudah untuk dipahami saat bekerja dengan banyak flag, terutama di sistem yang membutuhkan efisiensi penyimpanan sekaligus fleksibilitas dalam operasi bitwise.

### c. Operasi Bit Mask

#### Konsep Bit Mask

Bit mask adalah teknik yang digunakan untuk **mengetahui atau mengekstrak bit tertentu** dari sebuah bit string. Caranya adalah dengan membandingkan bit string tersebut dengan pola bit lain yang disebut _mask_. Mask ini menentukan bit mana yang ingin Anda periksa.
Biasanya, proses ini dilakukan menggunakan operasi **bitwise AND (`&`)**, karena AND memungkinkan kita melihat apakah bit tertentu dalam bit string sedang aktif (1) atau tidak (0).

#### Aturan Operasi AND (`&`)

Operasi AND bekerja dengan membandingkan setiap bit pada posisi yang sama di dua bit string. Aturannya sebagai berikut:

- Jika kedua bit bernilai **1**, hasilnya **1**
- Jika salah satu atau kedua bit bernilai **0**, hasilnya **0**

Aturan sederhana ini membuat operasi AND sangat cocok untuk memeriksa status bit tertentu.

#### Contoh Kasus: Feature Flags

Bayangkan Anda menyimpan status fitur yang diaktifkan oleh seorang pengguna. Misalnya:

- Feature 1: aktif
- Feature 2: tidak aktif
- Feature 3: aktif

Ini dapat direpresentasikan sebagai bit string:

```sql
-- Feature flags user: feature 1 dan 3 aktif
SELECT B'101' AS user_features;
-- Output: 101
```

Angka paling kiri adalah feature 1, angka tengah feature 2, dan angka kanan feature 3.

Sekarang kita ingin mengecek apakah masing-masing fitur aktif atau tidak.

```sql
-- Mask untuk cek feature 1
SELECT B'101' & B'100' AS check_feature_1;
-- Output: 100  → artinya feature 1 aktif

-- Mask untuk cek feature 2
SELECT B'101' & B'010' AS check_feature_2;
-- Output: 000  → artinya feature 2 tidak aktif

-- Mask untuk cek feature 3
SELECT B'101' & B'001' AS check_feature_3;
-- Output: 001  → artinya feature 3 aktif
```

Mask seperti `B'100'`, `B'010'`, dan `B'001'` menentukan bit mana yang sedang diperiksa. Jika hasil operasi AND menghasilkan angka selain `0`, berarti bit tersebut aktif.

#### Diagram Operasi AND

Berikut ilustrasi cara kerjanya:

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

Dari ilustrasi tersebut, Anda bisa melihat bahwa hanya bit yang sama-sama bernilai 1 yang menghasilkan 1.
Dengan memahami konsep bit mask dan operasi AND ini, Anda dapat mengelola banyak flag dalam satu kolom secara efisien dan tetap mudah diperiksa menggunakan operasi bitwise.

### d. Tipe Bit String: `BIT` vs `BIT VARYING`

#### Dua Jenis Tipe Bit String

PostgreSQL menyediakan dua tipe utama untuk menyimpan data berupa bit string, dan keduanya memiliki karakteristik yang berbeda tergantung kebutuhan Anda.

#### 1. `BIT(n)` — Panjang Tetap

Tipe `BIT(n)` digunakan ketika Anda ingin memastikan bahwa nilai yang disimpan **selalu memiliki panjang yang sama**, misalnya tepat 3 bit, 8 bit, atau 32 bit.
Jika Anda mencoba memasukkan bit string dengan panjang yang tidak sesuai dengan deklarasi, PostgreSQL akan mengembalikan error.
Tipe ini berguna untuk data yang struktur bit-nya sudah jelas dan tidak berubah-ubah.

#### 2. `BIT VARYING(n)` — Panjang Fleksibel

Berbeda dari `BIT(n)`, tipe `BIT VARYING(n)` (sering disingkat `VARBIT`) memungkinkan bit string memiliki panjang **beragam**, selama tidak melebihi batas yang Anda tetapkan.
Tipe ini lebih cocok untuk data yang flag-nya dinamis atau jumlah bit-nya tidak selalu konsisten.

#### Contoh Implementasi

Berikut contoh definisi tabel dan bagaimana kedua tipe ini bekerja:

```sql
CREATE TABLE bits (
  bit_fixed BIT(3),           -- harus tepat 3 bit
  bit_var BIT VARYING(32)     -- bisa 1–32 bit
);
```

Masukkan nilai yang valid:

```sql
INSERT INTO bits VALUES (
  B'001',                     -- tepat 3 bit ✓
  B'10101'                    -- 5 bit, masih di bawah 32 ✓
);
```

Coba insert dengan panjang bit yang salah:

```sql
INSERT INTO bits VALUES (
  B'01',                      -- hanya 2 bit ✗
  B'001'
);
-- ERROR: bit string length 2 does not match type bit(3)
```

Dan contoh insert lain yang valid:

```sql
INSERT INTO bits VALUES (
  B'001',                     -- tepat 3 bit ✓
  B'1'                        -- 1 bit, di bawah 32 ✓
);
```

Dari contoh di atas, terlihat jelas bahwa `BIT(n)` lebih ketat dan memastikan konsistensi, sedangkan `BIT VARYING(n)` memberikan fleksibilitas lebih besar untuk data yang panjang bit-nya tidak selalu sama tetapi tetap dikontrol dengan batas maksimum.

### e. Use Case Real-World

#### Contoh 1: User Permissions (8 Permissions)

Bit string sangat cocok untuk sistem yang memiliki banyak jenis permission tetapi ingin menyimpannya secara ringkas dalam satu kolom. Misalnya, Anda memiliki delapan jenis permission berbeda untuk setiap user. Alih-alih membuat delapan kolom boolean, Anda bisa menggunakan `BIT(8)` untuk menampung semuanya.

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username TEXT,
  permissions BIT(8)  -- 8 jenis permission
);
```

Misalnya, Anda mendefinisikan posisi bit sebagai berikut:

```
[0]: can_read
[1]: can_write
[2]: can_delete
[3]: can_admin
[4]: can_export
[5]: can_import
[6]: can_share
[7]: can_publish
```

Artinya, setiap bit mewakili izin tertentu. Jika bit-nya `1`, izin itu aktif; jika `0`, berarti tidak aktif.

Contoh data pengguna:

```sql
INSERT INTO users (username, permissions)
VALUES ('admin', B'11111111');  -- semua permission aktif

INSERT INTO users (username, permissions)
VALUES ('editor', B'11100000'); -- hanya read, write, delete
```

Untuk mengecek apakah seorang user memiliki permission tertentu, Anda bisa menggunakan bit mask. Misalnya, untuk mengecek apakah user memiliki permission **delete** (bit ke-2):

```sql
SELECT username,
       (permissions & B'00100000')::INT > 0 AS can_delete
FROM users;
```

Di sini, `permissions & B'00100000'` akan menghasilkan nilai bit yang hanya fokus pada posisi delete. Jika hasilnya bukan 0, berarti delete permission aktif.

#### Contoh 2: Days of Week (7 Days)

Bit string juga efektif untuk merepresentasikan kumpulan status yang berulang, seperti hari dalam seminggu. Anda dapat menggunakan 7 bit untuk mewakili Senin sampai Minggu.

```sql
CREATE TABLE events (
  id SERIAL PRIMARY KEY,
  name TEXT,
  active_days BIT(7)  -- Mon, Tue, Wed, Thu, Fri, Sat, Sun
);
```

Contoh event yang hanya aktif pada hari Senin, Rabu, dan Jumat:

```sql
INSERT INTO events (name, active_days)
VALUES ('Gym Class', B'1010100');  -- Mon, Wed, Fri
```

Jika Anda ingin mengecek apakah sebuah event aktif pada hari Senin, cukup gunakan mask yang sesuai dengan bit pertama:

```sql
SELECT name,
       (active_days & B'1000000')::INT > 0 AS active_on_monday
FROM events;
```

Seperti sebelumnya, jika hasilnya lebih besar dari 0, berarti event tersebut memang aktif pada hari Senin.

Dengan pendekatan ini, Anda dapat menyimpan banyak status dalam satu kolom dengan tetap mempertahankan kemampuan query yang efisien — sesuatu yang sulit dicapai jika menggunakan banyak kolom boolean atau struktur JSON.

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
