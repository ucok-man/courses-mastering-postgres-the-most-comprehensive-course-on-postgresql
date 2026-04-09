# 📝 JSON dan JSONB di PostgreSQL

## 1. Ringkasan Singkat

Materi ini membahas tentang menyimpan data **JSON** di PostgreSQL, topik yang cukup kontroversial di dunia database relational. PostgreSQL menyediakan dua tipe data untuk JSON: `JSON` dan `JSONB`. Video ini menjelaskan perbedaan keduanya, kapan sebaiknya menggunakan JSON di database, pedoman untuk memutuskan apakah data harus disimpan sebagai JSON atau dipecah menjadi kolom-kolom terpisah, serta preview singkat tentang operasi-operasi yang bisa dilakukan pada data JSON.

## 2. Konsep Utama

### a. Kontroversi: Haruskah JSON Disimpan di Database Relational?

Topik ini sering memicu perdebatan di kalangan developer, terutama saat menggunakan database seperti PostgreSQL. Intinya, ada dua pandangan yang cukup ekstrem tentang penggunaan JSON di database relasional.

#### Dua kubu ekstrem

Bayangkan ada dua kelompok dengan filosofi yang sangat berbeda:

- **Kubu 1 (Puritan)**
  Mereka berpendapat bahwa database relasional harus tetap “murni”. Artinya:
  - Data harus disimpan dalam bentuk tabel yang terstruktur
  - Setiap kolom punya tipe data yang jelas
  - Relasi antar tabel harus eksplisit

  Bagi mereka, menyimpan JSON di database relasional itu seperti “melanggar aturan”. JSON dianggap tidak cocok karena:
  - Strukturnya fleksibel (tidak ketat seperti skema tabel)
  - Sulit dijaga konsistensinya jika tidak dikontrol dengan baik

- **Kubu 2 (Liberal)**
  Di sisi lain, ada yang berpikir lebih fleksibel. Mereka melihat PostgreSQL bukan hanya sebagai database relasional, tapi juga bisa diperlakukan seperti NoSQL (misalnya seperti MongoDB).

  Pendekatannya:
  - Gunakan primary key biasa
  - Simpan sebagian besar data dalam bentuk JSON (sering disebut “JSON blob”)

  Keuntungannya:
  - Lebih fleksibel (tidak perlu ubah skema tiap kali struktur data berubah)
  - Cocok untuk data yang tidak konsisten atau sering berubah

#### Pendekatan tengah (yang direkomendasikan)

Daripada memilih salah satu ekstrem, pendekatan yang paling sehat adalah berada di tengah:

- JSON **bukan sesuatu yang salah**
- Tapi juga **bukan solusi untuk semua hal**

Anggap JSON sebagai **alat tambahan**, bukan pengganti total desain relasional.

#### Posisi yang direkomendasikan

Berikut prinsip yang lebih praktis dan realistis:

- JSON itu **sangat berguna**, terutama di PostgreSQL yang memang punya dukungan kuat seperti:
  - Query JSON (misalnya operator `->`, `->>`)
  - Indexing JSON (GIN index)
  - Update sebagian isi JSON tanpa overwrite seluruh data

- Tidak ada masalah menyimpan JSON, selama:
  - Kamu paham tujuan penggunaannya
  - Tidak mengorbankan struktur data yang seharusnya jelas

- Kunci utamanya adalah: **tahu kapan menggunakan JSON dan kapan menggunakan kolom biasa**

  Sebagai gambaran sederhana:
  - Gunakan **kolom terpisah** jika:
    - Data selalu ada dan strukturnya tetap (misalnya: nama, email, tanggal lahir)

  - Gunakan **JSON** jika:
    - Data opsional atau fleksibel (misalnya: preferensi user, metadata tambahan, konfigurasi dinamis)

Dengan pola pikir ini, kamu bisa memanfaatkan kekuatan database relasional tanpa kehilangan fleksibilitas yang ditawarkan JSON.

### b. Pedoman Kapan Menggunakan JSON vs Kolom Terpisah

Bagian ini membantu kamu mengambil keputusan yang lebih tepat: kapan sebaiknya memakai JSON, dan kapan lebih baik tetap menggunakan kolom biasa di database seperti PostgreSQL.

Pendekatannya bukan sekadar preferensi, tapi berdasarkan **bagaimana data digunakan (access pattern)**, **struktur skema**, dan **ukuran data**.

#### 1. Pertimbangan Access Pattern

Hal pertama yang perlu dipikirkan adalah: **bagaimana data akan diakses?**

##### ❌ Jangan gunakan JSON jika:

- Kamu **sering melakukan query** ke field tertentu di dalam JSON
- Field tersebut menjadi **kunci utama pencarian** (misalnya email user)

Kenapa ini penting?
Karena query ke dalam JSON biasanya:

- Lebih kompleks
- Bisa lebih lambat dibanding kolom biasa
- Sulit dioptimalkan jika sering digunakan

##### ✅ Gunakan JSON jika:

- Data biasanya diambil atau di-update **sebagai satu kesatuan (blob)**
- Jarang perlu mengakses field individual di dalamnya
- Struktur datanya **fleksibel atau semi-structured**

##### Contoh kasus

Perhatikan dua desain berikut:

```sql
-- ❌ KURANG OPTIMAL (email sering di-query)
CREATE TABLE users_bad (
    id BIGINT PRIMARY KEY,
    data JSONB  -- berisi: {email, name, address, preferences, ...}
);
```

Di sini, semua data user dimasukkan ke dalam satu kolom JSON.
Masalahnya: kalau kamu sering mencari user berdasarkan email, query jadi tidak efisien.

Bandingkan dengan:

```sql
-- ✅ LEBIH BAIK
CREATE TABLE users_good (
    id BIGINT PRIMARY KEY,
    email TEXT NOT NULL,              -- Top-level column
    name TEXT,                         -- Top-level column
    preferences JSONB                  -- JSON untuk data yang jarang di-query
);
```

Desain ini lebih optimal karena:

- Field penting seperti `email` dan `name` dibuat sebagai kolom biasa
- JSON hanya digunakan untuk data tambahan yang jarang diakses

##### Solusi tengah: Generated Columns

Kalau kamu tetap ingin menyimpan JSON lengkap:

- Simpan JSON sebagai sumber utama
- Tapi **ekstrak field penting ke kolom terpisah**

Ini memberi kamu fleksibilitas JSON + performa query kolom biasa. Konsep ini dikenal sebagai _generated columns_.

#### 2. Pertimbangan Schema Structure

Selanjutnya, lihat dari sisi **struktur data (schema)**.

##### Tanda harus dipisah ke kolom terpisah:

- Struktur data sudah **jelas dan stabil**
- Field-field sudah diketahui dari awal
- Setiap field sering di-update **secara independen**

Dalam kondisi ini, pendekatan relasional klasik lebih cocok.

##### Tanda cocok menggunakan JSON:

- Struktur data **fleksibel atau sering berubah**
- Tidak semua record punya field yang sama
- Biasanya di-update **sekaligus sebagai satu dokumen**

Selain itu, PostgreSQL juga mendukung update sebagian JSON (JSON patching), jadi tetap cukup powerful.

##### Contoh

Schema yang rigid:

```sql
-- Schema rigid → Pecah ke kolom
CREATE TABLE products_structured (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL,
    category TEXT
);
```

Semua field jelas dan konsisten → cocok pakai kolom biasa.

Schema yang fleksibel:

```sql
-- Schema flexible → Simpan sebagai JSON
CREATE TABLE products_flexible (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    metadata JSONB  -- attributes yang bervariasi per product type
);
```

Di sini, `metadata` bisa berbeda-beda tergantung jenis produk → JSON jadi solusi yang lebih praktis.

#### 3. Pertimbangan Ukuran Document

Terakhir, jangan lupa soal **ukuran data**.

##### Batasan teknis

- Kolom `JSONB` di PostgreSQL bisa menyimpan hingga sekitar **255 MB per dokumen**

Tapi penting diingat:
**“Bisa” tidak berarti “sebaiknya dilakukan.”**

##### Rekomendasi praktis

- Jika ukuran JSON sangat besar (misalnya puluhan MB):
  - Query akan lebih berat
  - Update bisa jadi mahal (costly)

- Lebih baik **pecah menjadi beberapa kolom JSON yang lebih kecil**

Dengan begitu:

- Akses data jadi lebih fokus (scoped)
- Performa lebih terjaga

##### Contoh

Desain kurang optimal:

```sql
-- ❌ Satu JSON blob besar (30 MB)
CREATE TABLE user_data_bad (
    user_id BIGINT PRIMARY KEY,
    all_data JSONB  -- profile + preferences + history + analytics
);
```

Semua data dimasukkan ke satu JSON besar → sulit di-manage dan berat diakses.

Desain yang lebih baik:

```sql
-- ✅ Dipecah ke beberapa JSON column
CREATE TABLE user_data_good (
    user_id BIGINT PRIMARY KEY,
    profile JSONB,        -- ~1 MB
    preferences JSONB,    -- ~500 KB
    activity_log JSONB,   -- ~10 MB
    analytics JSONB       -- ~5 MB
);
```

Dengan pemisahan ini:

- Kamu hanya mengambil data yang dibutuhkan
- Query lebih efisien
- Update lebih terkontrol

Dengan memahami tiga aspek ini—**access pattern, schema structure, dan ukuran data**—kamu bisa membuat keputusan desain yang lebih matang, tanpa terjebak ke salah satu ekstrem penggunaan JSON.

### c. JSON vs JSONB: Perbedaan Fundamental

Di PostgreSQL, ada dua tipe data untuk menyimpan JSON: **JSON** dan **JSONB**. Sekilas terlihat mirip, tapi sebenarnya cara kerja dan performanya cukup berbeda secara fundamental.

Supaya tidak salah pilih, kita perlu memahami bagaimana masing-masing tipe ini menyimpan dan memproses data.

#### Perbandingan JSON vs JSONB

Mari kita bahas perbedaannya secara konseptual, bukan sekadar tabel:

- **Storage**
  - JSON disimpan dalam bentuk **teks mentah (raw text)** → ukurannya biasanya lebih kecil
  - JSONB disimpan dalam bentuk **binary (sudah diproses)** → ukurannya lebih besar karena ada metadata tambahan

- **Parsing**
  - JSON: setiap kali diakses, harus di-parse ulang
  - JSONB: hanya di-parse sekali saat insert → setelah itu siap digunakan

- **Performance**
  - JSON: cenderung lebih lambat untuk query dan manipulasi
  - JSONB: jauh lebih cepat, karena sudah dalam format siap pakai

- **Whitespace dan formatting**
  - JSON mempertahankan spasi dan formatting asli
  - JSONB akan “menormalkan” data (menghapus whitespace yang tidak perlu)

- **Key order dan duplicate keys**
  - JSON mempertahankan urutan key dan duplicate keys
  - JSONB:
    - Tidak menjamin urutan key
    - Duplicate key akan di-overwrite (mengikuti standar JSON: key terakhir menang)

- **Indexing**
  - JSON tidak bisa di-index
  - JSONB bisa di-index (misalnya dengan GIN index) → ini krusial untuk performa

- **Use case**
  - JSON: biasanya hanya untuk kompatibilitas atau kebutuhan khusus
  - JSONB: **hampir selalu menjadi pilihan utama**

#### Contoh Perbedaan Ukuran Storage

Mari lihat bagaimana ukuran data berbeda antara JSON dan JSONB:

```sql
-- Contoh sederhana
SELECT
    pg_column_size('{"a": 1}'::JSON) as json_size,    -- 5 bytes
    pg_column_size('{"a": 1}'::JSONB) as jsonb_size;  -- 20 bytes
```

Hasilnya:

- JSON lebih kecil
- JSONB lebih besar karena ada struktur internal tambahan

Contoh lain dengan data lebih besar:

```sql
SELECT
    pg_column_size('{"a": "hello world"}'::JSON) as json_size,
    pg_column_size('{"a": "hello world"}'::JSONB) as jsonb_size;
```

Di sini, perbedaannya mulai mengecil. Kenapa?
Karena overhead JSONB menjadi relatif kecil dibanding total ukuran data (istilahnya: _overhead ter-amortisasi_).

##### Kenapa JSONB lebih besar?

Karena JSONB:

- Sudah di-parse saat disimpan
- Dipecah ke struktur internal (deconstructed)
- Disimpan dalam format binary
- Menyimpan metadata tambahan untuk mempercepat query

Semua ini menambah ukuran, tapi memberikan **keuntungan performa yang signifikan**.

#### Contoh Behavior: Whitespace

Perbedaan pertama yang mudah terlihat adalah cara menangani whitespace.

```sql
-- JSON: mempertahankan whitespace
SELECT '{"a":    "hello world"}'::JSON;
-- Output: {"a":    "hello world"}
```

JSON menyimpan teks persis seperti input.

```sql
-- JSONB: menghilangkan whitespace
SELECT '{"a":    "hello world"}'::JSONB;
-- Output: {"a":"hello world"}
```

JSONB akan menormalkan format, sehingga lebih konsisten.

#### Contoh Behavior: Duplicate Keys

Sekarang lihat bagaimana duplicate key diperlakukan:

```sql
-- JSON: mempertahankan duplicate keys
SELECT '{"a": 1, "a": 2}'::JSON;
-- Output: {"a": 1, "a": 2}
```

JSON tidak memproses isi—hanya menyimpan teks.

```sql
-- JSONB: key terakhir menang
SELECT '{"a": 1, "a": 2}'::JSONB;
-- Output: {"a":2}
```

JSONB mengikuti aturan umum JSON: jika ada key duplikat, nilai terakhir yang digunakan.

#### Contoh Behavior: Key Ordering

Terakhir, soal urutan key:

```sql
-- JSON: mempertahankan urutan
SELECT '{"z": 1, "a": 2, "m": 3}'::JSON;
-- Output: {"z": 1, "a": 2, "m": 3}
```

Urutan tetap sama seperti input.

```sql
-- JSONB: urutan tidak dijamin
SELECT '{"z": 1, "a": 2, "m": 3}'::JSONB;
-- Output: bisa berubah urutannya
```

JSONB tidak menjamin urutan, karena fokusnya adalah efisiensi akses, bukan representasi teks.

#### Kesimpulan Praktis

Kalau kamu bekerja di PostgreSQL:

- Gunakan **JSONB secara default**
- Gunakan JSON hanya jika:
  - Kamu butuh mempertahankan format asli (termasuk whitespace dan urutan)
  - Atau untuk kebutuhan kompatibilitas tertentu

Dalam hampir semua kasus nyata—terutama yang melibatkan query, indexing, dan performa—**JSONB adalah pilihan yang tepat**.

### d. Kapan Menggunakan JSON (bukan JSONB)?

Setelah memahami perbedaan antara JSON dan JSONB di PostgreSQL, pertanyaan berikutnya adalah: **apakah masih ada alasan untuk menggunakan JSON biasa?**

Jawaban singkatnya: **ada, tapi sangat jarang**. Mari kita bahas secara lebih terarah supaya kamu tahu kapan (dan kenapa) JSON masih relevan.

#### ⚠️ Use case JSON (sangat jarang)

Secara umum, JSON hanya digunakan dalam kondisi khusus yang memang membutuhkan sifat “mentah” dari data JSON.

##### 1. Legacy systems

Beberapa sistem lama (legacy) memiliki ketergantungan pada cara data JSON disimpan secara persis seperti input aslinya.

Contohnya:

- **Whitespace preservation**
  Sistem mengandalkan format teks JSON, termasuk spasi dan indentation.
  JSON mempertahankan ini, sedangkan JSONB akan menghapusnya.

- **Duplicate keys**
  Meskipun tidak direkomendasikan secara standar JSON, ada sistem yang tetap menggunakan key duplikat.
  JSON akan menyimpan semuanya, sementara JSONB hanya menyimpan key terakhir.

- **Key ordering**
  Ada kasus di mana urutan key dianggap penting (misalnya untuk hashing atau signature tertentu).
  JSON mempertahankan urutan, JSONB tidak menjaminnya.

Intinya, jika kamu bekerja dengan sistem lama yang sensitif terhadap representasi teks JSON secara persis, maka JSON bisa jadi diperlukan.

##### 2. Validasi tanpa operasi

Use case lain adalah ketika kamu hanya butuh JSON sebagai **validator format**, bukan sebagai data yang akan diolah.

Misalnya:

- Kamu hanya ingin memastikan input adalah JSON yang valid
- Data disimpan dan diambil kembali **tanpa perubahan sama sekali**
- Tidak ada kebutuhan untuk:
  - Query field di dalam JSON
  - Update sebagian isi JSON
  - Indexing

Dalam kasus ini, JSON cukup karena:

- Dia menyimpan data persis seperti input
- Tidak ada overhead parsing seperti JSONB

#### ✅ Rekomendasi utama

Untuk hampir semua use case modern di PostgreSQL, pilihan terbaik adalah:

- Gunakan **JSONB sebagai default**

Alasannya jelas:

- **Lebih cepat** untuk query dan manipulasi data
- Bisa dibuat **index** → sangat penting untuk performa
- Mengikuti **JSON specification** dengan lebih konsisten
- Mendukung berbagai fitur PostgreSQL seperti:
  - Query operator (`->`, `->>`)
  - Partial update
  - GIN index

#### Kesimpulan praktis

Anggap JSON sebagai opsi khusus untuk kebutuhan yang sangat spesifik, seperti kompatibilitas dengan sistem lama atau kebutuhan representasi teks mentah.

Di luar itu, untuk aplikasi modern—terutama yang butuh performa, fleksibilitas query, dan skalabilitas—**JSONB adalah pilihan yang hampir selalu tepat**.

### e. Preview: Operasi pada JSONB

Salah satu alasan utama kenapa JSONB sangat powerful di PostgreSQL adalah karena banyaknya operator dan fungsi yang bisa digunakan untuk mengakses dan memanipulasi datanya.

Di bagian ini, kita akan melihat beberapa operasi dasar yang paling sering digunakan. Anggap ini sebagai “preview” sebelum masuk ke pembahasan yang lebih dalam.

#### Ekstraksi Value dengan Operator `->`

Operator `->` digunakan untuk mengambil value dari JSONB, tetapi hasilnya **masih dalam bentuk JSON (bukan text biasa)**.

```sql
-- Ekstrak value (hasil masih JSONB)
SELECT '{"string": "hello world", "number": 42}'::JSONB -> 'string';
-- Output: "hello world"  (masih ada quotes, tipe JSONB)
```

Penjelasan:

- Kita mengambil key `"string"`
- Hasilnya `"hello world"` masih memiliki tanda kutip
- Ini karena tipe datanya tetap JSONB, bukan string biasa

Gunakan operator ini jika kamu masih ingin memproses hasilnya sebagai JSON.

#### Ekstraksi Text dengan Operator `->>`

Kalau kamu ingin hasil dalam bentuk **text biasa (tanpa tanda kutip)**, gunakan `->>`.

```sql
-- Ekstrak sebagai text (unquoted)
SELECT '{"string": "hello world", "number": 42}'::JSONB ->> 'string';
-- Output: hello world  (tanpa quotes, tipe TEXT)
```

Perbedaannya penting:

- `->` → hasil JSON
- `->>` → hasil TEXT

Biasanya `->>` lebih sering dipakai untuk kebutuhan query biasa (misalnya filtering atau ditampilkan ke user).

#### Akses Nested Objects

JSON sering memiliki struktur bertingkat (nested). Kamu bisa mengaksesnya dengan chaining operator `->`.

```sql
SELECT '{
  "objects": {
    "key": "value"
  },
  "array": [1, 2, 3]
}'::JSONB -> 'objects' -> 'key';
-- Output: "value"
```

Penjelasan:

- Ambil `objects`
- Lalu ambil `key` di dalamnya
- Hasil tetap dalam format JSONB (`"value"`)

Ini seperti menavigasi object di JavaScript: `obj.objects.key`

#### Akses Array Elements

JSONB juga mendukung array, dan kamu bisa mengakses elemennya dengan index.

```sql
-- Menggunakan index array
SELECT '{
  "array": [10, 20, 30]
}'::JSONB -> 'array' -> 2;
-- Output: 30  (index dimulai dari 0)
```

Catatan penting:

- Index dimulai dari **0**, bukan 1
- Jadi:
  - `0` → 10
  - `1` → 20
  - `2` → 30

#### Path Expressions dengan `#>` Operator

Kalau struktur JSON cukup dalam, chaining `->` bisa jadi panjang.
Di sinilah operator `#>` berguna, karena kita bisa langsung menentukan path.

```sql
-- Akses nested structure dengan path
SELECT '{
  "objects": {
    "nested": {
      "key": "deep value"
    }
  }
}'::JSONB #> '{objects,nested,key}';
-- Output: "deep value"
```

Penjelasan:

- Path ditulis sebagai array: `{objects,nested,key}`
- PostgreSQL akan langsung menelusuri struktur tersebut

Ini lebih ringkas dan sering lebih nyaman untuk struktur yang kompleks.

#### Operasi lain yang akan dibahas

Selain operator dasar di atas, PostgreSQL masih punya banyak fitur lanjutan untuk JSONB, seperti:

- Mengecek apakah suatu key ada
- Mengecek overlap antar struktur JSON
- Operasi pada array (filter, expand, dll)
- Update sebagian isi JSON (JSON patching)
- Strategi indexing untuk performa tinggi

Semua ini membuat JSONB bukan sekadar tempat menyimpan data fleksibel, tapi juga alat yang sangat kuat untuk query dan manipulasi data semi-structured langsung di dalam database.

### f. Best Practices untuk JSON di PostgreSQL

Setelah memahami kapan menggunakan JSON, JSONB, dan kolom biasa, sekarang kita rangkum dalam bentuk **panduan praktis** yang bisa kamu pakai saat mendesain schema di PostgreSQL.

Tujuannya sederhana: membantu kamu mengambil keputusan dengan cepat tanpa harus menebak-nebak.

#### Checklist keputusan

Bayangkan kamu sedang menentukan apakah sebuah field sebaiknya disimpan sebagai JSONB atau kolom biasa. Gunakan alur berpikir berikut:

- **Apakah field ini sering di-query?**
  Jika _ya_, maka:
  - Simpan sebagai **top-level column**

  Kenapa?
  Karena kolom biasa lebih cepat untuk filtering, indexing, dan lookup.

- **Jika tidak sering di-query → lanjutkan**
  Tanyakan: apakah struktur datanya **rigid dan well-defined**?
  - Jika _ya_:
    - Lebih baik gunakan **kolom terpisah**

  Artinya, struktur sudah jelas dan stabil → tidak perlu fleksibilitas JSON.

- **Jika schema tidak rigid → lanjutkan**
  Tanyakan: apakah field sering di-update secara independen?
  - Jika _ya_:
    - Pertimbangkan **kolom terpisah**

  Karena update parsial pada JSON bisa lebih kompleks dibanding update kolom biasa.

- **Jika tidak → lanjutkan**
  Tanyakan: apakah ukuran document sangat besar (misalnya >10MB)?
  - Jika _ya_:
    - **Jangan simpan dalam satu JSON besar**
    - Pecah menjadi beberapa JSON column

  Ini penting untuk menjaga performa tetap optimal.

- **Jika semua kondisi di atas tidak terpenuhi**
  Maka:
  - Gunakan **JSONB** sebagai solusi fleksibel

Dengan pola ini, kamu tidak akan “asal pakai JSON”, tapi benar-benar berdasarkan kebutuhan.

#### Golden Rules

Sebagai ringkasan, berikut prinsip-prinsip yang sebaiknya kamu pegang saat bekerja dengan JSON di PostgreSQL:

1. **Gunakan JSONB (bukan JSON) untuk hampir semua kasus**
   JSONB memberikan performa lebih baik, bisa di-index, dan lebih powerful untuk query.

2. **Gunakan top-level columns untuk data yang sering di-query**
   Misalnya: email, username, status, atau field yang sering dipakai di WHERE clause.

3. **Gunakan JSON columns untuk data yang fleksibel atau semi-structured**
   Cocok untuk:
   - Preferences user
   - Metadata tambahan
   - Data yang tidak selalu konsisten strukturnya

4. **Perhatikan ukuran document**
   Jangan biarkan satu JSON menjadi terlalu besar:
   - Sulit di-query
   - Berat saat di-update
   - Bisa menurunkan performa

5. **PostgreSQL bekerja paling optimal dengan typed columns yang jelas**
   Artinya:
   - Gunakan tipe data yang spesifik jika memungkinkan
   - Gunakan JSON sebagai pelengkap, bukan pengganti total desain relasional

Dengan mengikuti best practices ini, kamu bisa memanfaatkan fleksibilitas JSON tanpa mengorbankan kekuatan utama database relasional seperti PostgreSQL.

## 3. Hubungan Antar Konsep

Filosofi penyimpanan data di PostgreSQL:

```
Prinsip Fundamental:
"Pick the data type that most accurately represents reality"
                    |
        ┌───────────┴───────────┐
        |                       |
   Structured Data        Semi-structured Data
   (Known schema)         (Flexible schema)
        |                       |
   ┌────┴────┐            ┌────┴────┐
   |         |            |         |
Discrete   JSON for   JSONB for  Legacy
Columns   metadata   flexibility  (JSON)
```

**Alur keputusan:**

1. **Mulai dengan struktur relational** → Kolom-kolom typed yang discrete
2. **Identifikasi data flexible** → Kandidat untuk JSON
3. **Evaluasi access patterns** → Sering di-query? → Extract ke top-level
4. **Pilih JSONB** (bukan JSON) untuk modern use cases
5. **Index jika perlu** → PostgreSQL support indexing JSONB

**Trade-offs:**

```
JSON Storage:
   Advantages                      Disadvantages
   ✅ Flexibility                  ❌ Kurang performant untuk frequent queries
   ✅ Schema evolution             ❌ Kehilangan type safety
   ✅ Nested structures            ❌ Kompleksitas query meningkat
   ✅ Compact untuk some data      ❌ Indexing lebih kompleks

Top-level Columns:
   Advantages                      Disadvantages
   ✅ Performance optimal          ❌ Less flexible
   ✅ Type safety                  ❌ Schema changes perlu migration
   ✅ Simple queries               ❌ Sulit untuk nested data
   ✅ Easy indexing                ❌ Banyak NULL jika data sparse
```

**Skenario ideal untuk JSONB:**

- User preferences/settings
- Product attributes (varies by category)
- Metadata/tags
- API responses yang disimpan
- Audit logs
- Configuration data

**Skenario ideal untuk kolom terpisah:**

- Primary identifiers (email, username)
- Frequently queried fields
- Fields yang di-update independen
- Foreign keys
- Core business data

## 4. Kesimpulan

- PostgreSQL menyediakan **dua tipe data JSON**: `JSON` dan `JSONB`
- **JSONB adalah pilihan yang direkomendasikan** untuk hampir semua use case:
  - Stored dalam **binary format** (sudah parsed)
  - **Lebih cepat** untuk operasi query/update
  - **Bisa di-index** untuk performance
  - Mengikuti **JSON spec** dengan benar (no duplicate keys)
- **JSON type** hanya untuk legacy compatibility (preserves whitespace, key order, duplicates)
- **Pedoman keputusan** menyimpan sebagai JSON vs kolom terpisah:
  1. **Access patterns** → Sering di-query? → Top-level column
  2. **Schema rigidity** → Rigid & well-defined? → Kolom terpisah
  3. **Update patterns** → Updated independently? → Kolom terpisah
  4. **Document size** → Sangat besar? → Pecah ke multiple columns
- PostgreSQL memiliki **fitur lengkap** untuk manipulasi JSON:
  - Extraction operators (`->`, `->>`, `#>`)
  - Path expressions
  - Indexing support
  - JSON patching
  - Array operations
- **Best practice**: Kombinasikan typed columns untuk data core dengan JSONB untuk flexibility

**Template yang direkomendasikan:**

```sql
CREATE TABLE modern_table (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

    -- Core fields: typed columns
    email TEXT NOT NULL,
    username TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Flexible data: JSONB
    preferences JSONB,
    metadata JSONB,

    -- Indexes untuk JSONB jika perlu
    -- (akan dibahas di module indexing)
);
```
