# 📝 Full Text Search di PostgreSQL: TS Vector dan TS Query

## 1. Ringkasan Singkat

Video ini membahas cara menggunakan fitur **full text search** bawaan PostgreSQL tanpa perlu tools eksternal seperti Elasticsearch. Fokus pembahasan adalah dua tipe data khusus: **`tsvector`** dan **`tsquery`**, serta cara mengimplementasikannya dengan **generated columns** agar data pencarian selalu sinkron dengan data asli.

---

## 2. Konsep Utama

### a. Full Text Search di PostgreSQL

#### Gambaran Umum

PostgreSQL memiliki fitur full text search bawaan yang cukup kuat dan dapat memenuhi sekitar **80–90% kebutuhan pencarian teks** dalam aplikasi. Artinya, untuk kebanyakan kasus seperti mencari judul artikel, deskripsi produk, atau konten sederhana, kita tidak perlu menambahkan sistem pencarian eksternal yang kompleks.

Keuntungan utama dari pendekatan ini adalah **arsitektur aplikasi tetap sederhana**. Kita hanya menggunakan satu database tanpa menambah service lain seperti Elasticsearch atau MeiliSearch. Ini bisa mengurangi biaya, beban operasional, serta mempermudah deployment.

#### Kapan Perlu Solusi Eksternal?

Meskipun powerful, full text search bawaan PostgreSQL tetap memiliki batasan. Jika aplikasi kita membutuhkan fitur pencarian yang lebih canggih—misalnya:

- **Faceted search** (filter dinamis berdasarkan kategori, tag, range harga, dan seterusnya)
- **Highlighting hasil** (menyorot kata yang cocok dalam hasil pencarian)
- Ranking yang lebih kompleks atau analisis linguistik mendalam

maka kita biasanya perlu menggunakan sistem khusus seperti Elasticsearch.

Beberapa penyedia hosting seperti Zeta bahkan menawarkan **replikasi otomatis** dari PostgreSQL ke Elasticsearch. Dengan cara ini, kita tetap menulis data ke Postgres, lalu Elasticsearch digunakan khusus untuk pencarian tingkat lanjut.

#### Contoh Penggunaan Sederhana

Misalnya kita memiliki tabel `articles` dengan kolom `title` dan `content`. Untuk melakukan full text search, kita bisa membuat indeks dan menjalankan query seperti berikut:

```sql
-- Membuat indeks full text search
CREATE INDEX articles_search_idx
ON articles
USING GIN (to_tsvector('english', title || ' ' || content));

-- Query pencarian
SELECT id, title
FROM articles
WHERE to_tsvector('english', title || ' ' || content)
      @@ plainto_tsquery('english', 'database scaling');
```

Pada query di atas:

- `to_tsvector` mengubah teks menjadi format yang bisa dicari oleh mesin FTS PostgreSQL.
- `plainto_tsquery` mengubah input user menjadi query yang aman.
- Operator `@@` digunakan untuk mencocokkan vektor teks dengan query.

Hasilnya akan berisi semua artikel yang relevan dengan pencarian _"database scaling"_.

Dengan fitur ini, kita bisa membangun pencarian yang cukup efisien tanpa tambahan service eksternal, selama kebutuhan masih dalam batas kemampuan PostgreSQL.

### b. Tipe Data `tsvector`

#### Apa Itu `tsvector`?

`tsvector` adalah tipe data khusus di PostgreSQL yang dirancang untuk kebutuhan full text search. Di dalamnya, PostgreSQL menyimpan **daftar lexeme**—yaitu bentuk dasar dari kata—yang sudah dibuat unik dan diurutkan secara alfabetis. Dengan struktur ini, proses pencarian menjadi lebih efisien karena database tidak perlu menganalisis teks mentah setiap kali query dijalankan.

#### Mengenal Lexeme

Lexeme adalah **unit dasar bahasa** yang sudah dinormalisasi. Normalisasi ini biasanya melibatkan proses stemming, yaitu pengubahan kata ke bentuk dasarnya.

Misalnya:

- "lazy" dan "laziness" akan dipetakan menjadi lexeme yang sama, yaitu `lazi`.
- Dengan cara ini, pencarian akan lebih fleksibel, karena secara otomatis mendukung **fuzzy matching** tanpa konfigurasi tambahan.

#### Membuat `tsvector`

Untuk mengubah teks biasa menjadi `tsvector`, kita menggunakan fungsi `to_tsvector`. Contoh sederhana:

```sql
SELECT to_tsvector('the quick brown fox');
```

Hasilnya:

```
'brown':3 'fox':4 'quick':2
```

Penjelasannya:

- Kata-kata diurutkan alfabetis: `brown`, `fox`, `quick`.
- Angka setelah titik dua menunjukkan **posisi kata dalam teks asli**.
- Kata seperti "the" dihilangkan karena dianggap **stop word**.

#### Kata yang Sama Muncul Lebih dari Sekali

Jika ada kata yang muncul berkali-kali, `tsvector` tetap menyimpannya sebagai **satu lexeme**, tetapi posisinya akan ditambahkan sebagai daftar.

Contoh:

```sql
SELECT to_tsvector('the quick brown fox jumps over the lazy fox');
```

Hasil:

```
'fox':4,9
```

Penjelasan:

- Hanya satu lexeme `fox` yang disimpan.
- Angka 4 dan 9 menunjukkan bahwa kata tersebut muncul pada posisi ke-4 dan ke-9.

#### Contoh Normalisasi Lexeme

Berikut contoh bagaimana kata yang bentuknya berbeda bisa distem menjadi lexeme yang sama:

```sql
SELECT to_tsvector('lazy laziness');
```

Hasil:

```
'lazi':1,2
```

Di sini:

- `lazy` dan `laziness` di-stem menjadi `lazi`.
- `tsvector` menyimpan satu lexeme `lazi` dengan dua posisi: 1 dan 2.

Dengan memahami cara kerja `tsvector`, kita dapat melihat mengapa PostgreSQL sangat efisien dalam melakukan full text search—teks sudah dalam bentuk terstruktur yang siap dicari tanpa proses parsing ulang.

### c. Tipe Data `tsquery`

#### Apa Itu `tsquery`?

`tsquery` adalah tipe data di PostgreSQL yang digunakan untuk menyusun **query pencarian** dalam full text search. Jika `tsvector` adalah bentuk terstruktur dari teks yang ingin dicari, maka `tsquery` adalah bentuk terstruktur dari teks atau kata kunci yang digunakan oleh pengguna saat melakukan pencarian.

Sama seperti `tsvector`, nilai di dalam `tsquery` juga akan **dinormalisasi menjadi lexeme**. Artinya, kata-kata yang bentuknya berbeda tetapi memiliki akar yang sama akan dianggap setara. Hal ini memungkinkan pencarian menjadi lebih fleksibel dan tidak terlalu sensitif terhadap variasi kata.

#### Membuat `tsquery`

Untuk membuat `tsquery`, kita menggunakan fungsi `to_tsquery`. Contoh paling sederhana adalah memasukkan satu kata:

```sql
SELECT to_tsquery('lazy');
```

Hasil:

```
'lazi'
```

Penjelasan:

- Kata **lazy** distem menjadi lexeme `lazi`.
- Inilah bentuk yang akan dicocokkan dengan `tsvector`.

Contoh lain:

```sql
SELECT to_tsquery('laziness');
```

Hasil:

```
'lazi'
```

Di sini:

- Meskipun kata yang diberikan berbeda (`lazy` vs `laziness`), PostgreSQL menormalisasinya menjadi lexeme yang sama.
- Hal ini menunjukkan bahwa `tsquery` mendukung pencarian berdasarkan akar kata, sehingga pengguna tetap bisa menemukan hasil yang relevan meskipun bentuk kata yang mereka masukkan berbeda dari teks aslinya.

Dengan memahami cara kerja `tsquery`, kita bisa menyusun query pencarian yang lebih akurat dan memanfaatkan sepenuhnya kemampuan full text search PostgreSQL.

### d. Operator Pencarian: `@@`

#### Fungsi Utama Operator `@@`

Operator `@@` adalah pusat dari proses full text search di PostgreSQL. Operator ini digunakan untuk **mencocokkan** sebuah `tsquery` (query pencarian) dengan sebuah `tsvector` (representasi teks yang telah dinormalisasi). Dengan kata lain, operator ini menjawab pertanyaan: _“Apakah kata atau frasa yang dicari muncul dalam teks ini?”_

Sintaks dasarnya sangat sederhana:

```sql
tsvector @@ tsquery
```

Jika kecocokan ditemukan, hasilnya `true`. Jika tidak, hasilnya `false`.

#### Contoh Penggunaan Dasar

Mari kita lihat bagaimana operator ini bekerja dalam beberapa situasi.

##### 1. Kata yang Dicari Ada dalam Teks

```sql
SELECT to_tsvector('the quick brown fox jumps over the lazy dog')
       @@ to_tsquery('lazy');
```

Output:

```
true
```

Penjelasan:

- Teks mengandung kata _lazy_.
- Setelah dinormalisasi menjadi `lazi`, lexeme tersebut cocok dengan query.

##### 2. Bentuk Kata Berbeda, Tapi Akar Kata Sama

```sql
SELECT to_tsvector('the quick brown fox jumps over the lazy dog')
       @@ to_tsquery('laziness');
```

Output:

```
true
```

Di sini:

- Teks aslinya hanya berisi “lazy”.
- Query berisi “laziness”.
- Keduanya distem menjadi lexeme yang sama, yaitu `lazi`, sehingga PostgreSQL menganggapnya cocok.

Ini adalah salah satu kekuatan utama full text search PostgreSQL—kemampuan melakukan pencarian berbasis akar kata secara otomatis.

##### 3. Kata Tidak Ada dalam Teks

```sql
SELECT to_tsvector('the quick brown fox jumps over the lazy dog')
       @@ to_tsquery('red');
```

Output:

```
false
```

Karena teks tidak mengandung lexeme `red`, pencarian tidak menghasilkan kecocokan.

#### Catatan Tambahan

- Urutan penulisan tidak memengaruhi hasil. Anda bisa menulis `tsquery @@ tsvector`, dan hasilnya tetap sama.
- PostgreSQL juga menyediakan berbagai operator tambahan seperti **AND**, **OR**, **NOT**, serta fitur penilaian relevansi (weighting dan ranking). Operator-operator ini akan dibahas lebih detail dalam modul lanjutan.

Dengan memahami operator `@@`, Anda sudah menguasai fondasi utama dari full text search di PostgreSQL. Operator inilah yang menghubungkan antara teks yang sudah diproses dengan kata kunci yang dicari pengguna.

### e. Konfigurasi Bahasa

#### Mengapa Bahasa Penting dalam Full Text Search?

Saat melakukan full text search, PostgreSQL tidak hanya memecah teks menjadi kata-kata, tetapi juga menerapkan aturan linguistik seperti **stemming** (mengubah kata ke bentuk dasarnya) dan **stop words** (mengabaikan kata-kata umum seperti “the”, “and”, atau “of”). Karena setiap bahasa memiliki aturan berbeda, PostgreSQL memungkinkan kita menentukan bahasa yang ingin digunakan saat memproses teks.

Fungsi `to_tsvector()` dapat menerima parameter bahasa agar proses analisis kata mengikuti aturan yang tepat.

#### Contoh Penggunaan Bahasa

```sql
SELECT to_tsvector('english', 'the quick brown fox');
SELECT to_tsvector('french', 'bonjour');
```

Pada contoh ini:

- Baris pertama menggunakan konfigurasi bahasa Inggris, sehingga stop words seperti “the” otomatis dihilangkan.
- Baris kedua menggunakan bahasa Prancis. Di sini, PostgreSQL menggunakan aturan stemming dan stop words khusus bahasa Prancis.

Dengan demikian, pemilihan bahasa yang tepat membuat hasil pencarian lebih akurat — terutama jika aplikasi Anda mendukung banyak bahasa.

#### Kenapa Pengaturan Bahasa Ini Penting?

Ada beberapa alasan yang perlu dipahami:

1. **Setiap bahasa memiliki aturan linguistiknya sendiri.**
   Misalnya, bahasa Inggris memiliki pola stemming berbeda dari bahasa Prancis atau Spanyol. Jika salah memilih bahasa, hasil pencarian bisa menjadi kurang relevan.

2. **PostgreSQL memiliki konfigurasi default.**
   Jika Anda tidak menentukan bahasa secara eksplisit, PostgreSQL akan menggunakan nilai dari `default_text_search_config`, yang biasanya diset ke bahasa Inggris.
   Ini bisa jadi masalah jika teks Anda bukan bahasa Inggris.

3. **Generated columns harus bersifat immutable.**
   Ketika Anda membuat kolom ter-generate (misalnya kolom `tsvector` otomatis yang dipakai untuk indeks), fungsi yang digunakan harus **immutable**.

   Agar `to_tsvector()` dianggap immutable, Anda **wajib** menentukan bahasa secara eksplisit. Jika Anda menuliskannya tanpa parameter bahasa, PostgreSQL menganggap hasilnya bisa berubah tergantung konfigurasi global, sehingga tidak immutable.

Contoh yang benar untuk generated column:

```sql
to_tsvector('english', content)
```

Dengan menentukan bahasa secara eksplisit, PostgreSQL bisa menjamin bahwa nilai yang dihasilkan tidak berubah-ubah, sehingga aman digunakan dalam indeks dan kolom ter-generate.

Dengan memahami konfigurasi bahasa ini, Anda bisa memastikan sistem pencarian bekerja konsisten dan akurat sesuai bahasa yang digunakan dalam data Anda.

### f. Implementasi dengan Generated Columns

#### Mengapa Kita Membutuhkan Generated Column?

Ketika Anda melakukan full text search, biasanya Anda menyimpan hasil `tsvector` dalam sebuah kolom khusus. Namun, jika kolom tersebut diisi secara manual (misalnya lewat trigger atau proses terpisah), ada risiko besar nilai `tsvector` tersebut **tidak lagi sinkron** dengan kolom teks aslinya. Artinya, ketika `content` berubah, kadang `tsvector` lupa diperbarui. Hal ini bisa membuat pencarian menjadi tidak akurat.

PostgreSQL menyediakan solusi yang lebih aman dan rapi: **generated columns**. Dengan ini, kolom `tsvector` akan dihitung ulang secara otomatis setiap kali `content` berubah. Anda tidak perlu membuat trigger, dan tidak ada risiko data tidak konsisten.

#### Contoh Tabel dengan Generated Column

```sql
CREATE TABLE ts_example (
  id SERIAL PRIMARY KEY,
  content TEXT,
  search_vector_en TSVECTOR
    GENERATED ALWAYS AS (to_tsvector('english', content)) STORED
);
```

Mari kita bahas bagian-bagian pentingnya:

##### `GENERATED ALWAYS AS`

Instruksi ini memberitahu PostgreSQL bahwa nilai kolom `search_vector_en` **tidak boleh diisi manual**. PostgreSQL akan selalu menghitungnya berdasarkan ekspresi yang diberikan.

##### `to_tsvector('english', content)`

Di sinilah proses konversi dilakukan. Setiap kali `content` berubah, PostgreSQL akan:

1. mengambil teks dari kolom `content`
2. menerapkannya ke fungsi `to_tsvector`
3. menyimpannya dalam kolom `search_vector_en`

Penggunaan `'english'` sebagai parameter bahasa **wajib**, karena tanpa ini fungsi dianggap tidak immutable (akan dijelaskan sebentar lagi).

##### `STORED`

Hasil ekspresi disimpan secara fisik di disk. PostgreSQL saat ini tidak mendukung generated column bertipe _virtual_, jadi penggunaan `STORED` adalah satu-satunya opsi.

#### Mengapa Harus Menentukan Bahasa Secara Eksplisit?

Ini terkait dengan sifat _immutability_. Generated column hanya boleh menggunakan fungsi yang **immutable**, yaitu fungsi yang selalu menghasilkan output yang sama untuk input yang sama.

Masalahnya:

- Jika Anda menulis `to_tsvector(content)` tanpa parameter bahasa, PostgreSQL menggunakan `default_text_search_config`.
- Konfigurasi default ini bisa berubah sewaktu-waktu oleh admin database.
- Artinya: input yang sama bisa menghasilkan output berbeda setelah konfigurasi berubah → fungsi menjadi **non-deterministic**.

Karena itu PostgreSQL akan menolak generated column seperti:

```sql
GENERATED ALWAYS AS (to_tsvector(content)) STORED
```

dan meminta Anda menentukan bahasa secara eksplisit:

```sql
to_tsvector('english', content)
```

Dengan menentukan bahasa, hasil fungsi menjadi deterministik, sehingga aman untuk digunakan dalam generated columns.

Generated columns adalah cara yang paling robust dan recommended untuk menyimpan `tsvector`, memastikan index dan pencarian selalu konsisten tanpa intervensi manual atau trigger tambahan.

### g. Query dengan Generated Column

#### Menambahkan Data ke Tabel

Setelah tabel dengan generated column dibuat, Anda cukup memasukkan nilai ke kolom `content`. PostgreSQL akan otomatis menghitung dan mengisi `search_vector_en` berdasarkan teks tersebut.

Contoh insert:

```sql
INSERT INTO ts_example (content)
VALUES ('the quick brown fox jumps over the lazy dog');

INSERT INTO ts_example (content)
VALUES ('the quick brown fox jumps over the cat');
```

Pada titik ini:

- Baris pertama memiliki lexeme seperti `quick`, `brown`, `fox`, `lazi`, dll.
- Baris kedua mirip, tetapi tidak memiliki `lazi`; sebagai gantinya ada lexeme `cat`.

Anda tidak perlu memasukkan apa pun ke kolom `search_vector_en` karena PostgreSQL mengisinya otomatis melalui ekspresi generated column.

#### Melakukan Pencarian Menggunakan `@@`

Sekarang, mari kita lihat bagaimana kita bisa memanfaatkan kolom `search_vector_en` untuk mencari baris tertentu.

##### Mencari Baris yang Mengandung Kata “lazy”

```sql
SELECT * FROM ts_example
WHERE search_vector_en @@ to_tsquery('lazy');
```

Hasil yang muncul:

- Hanya baris pertama.

Alasannya sederhana: hanya baris pertama yang memiliki kata _lazy_ (yang distem menjadi lexeme `lazi`). Generated column memastikan nilai `search_vector_en` sudah berisi lexeme tersebut.

##### Mencari Baris yang Mengandung Kata “cat”

```sql
SELECT * FROM ts_example
WHERE search_vector_en @@ to_tsquery('cat');
```

Hasil:

- Hanya baris kedua.

Dalam baris kedua, lexeme `cat` ada dalam `content`, sehingga query mencocokkannya.

#### Keuntungan Pendekatan Ini

Menggunakan generated column untuk menyimpan `tsvector` memberi Anda beberapa keuntungan besar:

- Anda hanya perlu melakukan **INSERT atau UPDATE pada kolom `content`**; kolom `search_vector_en` akan selalu otomatis diperbarui.
- Tidak perlu membuat trigger manual atau khawatir lupa memperbarui kolom pencarian.
- Tidak mungkin terjadi kondisi **out-of-sync** antara teks asli dan nilai `tsvector`.
- Query menggunakan operator `@@` menjadi sederhana dan konsisten.

Dengan konfigurasi seperti ini, sistem full text search Anda menjadi lebih andal, bersih, dan minim maintenance.

## 3. Hubungan Antar Konsep

```
┌─────────────┐
│ Text Asli   │ (disimpan di kolom TEXT)
└──────┬──────┘
       │
       │ to_tsvector('english', content)
       ▼
┌─────────────┐
│  tsvector   │ (generated column, normalized & indexed)
└──────┬──────┘
       │
       │ operator @@
       ▼
┌─────────────┐
│  tsquery    │ (query yang juga di-normalize)
└─────────────┘
       │
       ▼
  [true/false]
```

**Alur kerja:**

1. User menyimpan teks asli di kolom `content`
2. PostgreSQL otomatis mengkonversi ke `tsvector` via generated column
3. Saat search, query user dikonversi ke `tsquery`
4. Operator `@@` mencocokkan `tsvector` dengan `tsquery`
5. Return `true` jika match, `false` jika tidak

**Keuntungan pendekatan ini:**

- **Single source of truth**: hanya kolom `content` yang di-update manual
- **Always in sync**: `tsvector` pasti sesuai dengan `content`
- **Performance**: `tsvector` bisa di-index untuk pencarian cepat
- **Fuzzy matching gratis**: stemming otomatis (lazy = laziness)

---

## 4. Kesimpulan

PostgreSQL menyediakan fitur full text search yang powerful lewat dua tipe data khusus:

- **`tsvector`**: representasi teks yang sudah dinormalisasi dan dioptimalkan untuk pencarian
- **`tsquery`**: query pencarian yang juga dinormalisasi

Dengan kombinasi **generated columns**, kita bisa memastikan data pencarian selalu sinkron tanpa perlu trigger atau maintenance manual. Fitur ini mencakup 80-90% kebutuhan full text search tanpa perlu menambah kompleksitas infrastruktur.

**Next steps:**

- Pelajari operator advanced (AND, OR, NOT)
- Ranking dan weighting hasil pencarian
- Stop words dan konfigurasi bahasa
- Indexing strategy untuk performa optimal
