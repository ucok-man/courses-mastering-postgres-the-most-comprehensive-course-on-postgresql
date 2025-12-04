# 📝 Foreign Key Constraint di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang Foreign Key Constraint di PostgreSQL, termasuk perbedaan penting antara "foreign key" (konsep) dan "foreign key constraint" (enforcement). Materi mencakup cara membuat foreign key constraint, naming conventions, composite foreign keys, dan berbagai opsi untuk menangani operasi UPDATE dan DELETE pada parent table (ON DELETE/UPDATE actions).

## 2. Konsep Utama

### a. Foreign Key vs Foreign Key Constraint - Perbedaan Fundamental

#### Memahami Perbedaannya Secara Intuitif

Bagian ini membahas dua istilah yang sering dianggap sama, padahal sebenarnya berbeda secara konsep dan fungsi: **Foreign Key** dan **Foreign Key Constraint**. Perbedaannya penting karena berpengaruh langsung pada desain database, khususnya bagaimana kita menjaga konsistensi data.

Tabel berikut merangkum perbedaan mendasar keduanya:

| Aspek       | Foreign Key                        | Foreign Key Constraint             |
| ----------- | ---------------------------------- | ---------------------------------- |
| Definisi    | Konsep/filosofis                   | Implementasi/enforcement           |
| Sifat       | Hanya pointer/reference            | Aturan yang di-enforce database    |
| Enforcement | Tidak ada                          | Ada validasi otomatis              |
| Keharusan   | Selalu ada dalam relational design | Optional (bisa dipilih atau tidak) |

#### Apa Itu Foreign Key (Sebagai Konsep)?

Foreign key dalam konteks ini bukanlah “constraint”. Ia adalah **sekadar kolom biasa** yang menyimpan ID dari tabel lain untuk menunjukkan adanya relasi. Dengan kata lain, foreign key adalah konsep relasional yang menjelaskan bagaimana baris pada sebuah tabel terhubung dengan baris di tabel lain.

Beberapa karakteristik penting:

- Biasanya berupa sebuah kolom (atau beberapa kolom) yang menyimpan primary key dari tabel lain.
- Tidak membutuhkan constraint, index, atau aturan khusus apa pun.
- Selalu muncul ketika sebuah tabel memiliki hubungan dengan tabel lain, karena aplikasi pasti menyimpan ID referensi tersebut.
- Tidak ada validation otomatis dari database — sehingga database tidak peduli apakah nilai yang disimpan benar-benar ada di tabel parent.

Contoh:

```sql
-- Ini adalah foreign key (sebagai konsep)
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT  -- Kolom ini menyimpan ID dari tabel states
);
```

Pada contoh di atas, `state_id` sudah menjadi foreign key **secara konsep**, karena ia menunjuk ke “calon parent table” yaitu `states`. Namun, database **belum menerapkan aturan apa pun** untuk memastikan bahwa setiap `state_id` valid.

#### Apa Itu Foreign Key Constraint (Sebagai Enforcement)?

Berbeda dengan konsep foreign key, **foreign key constraint** adalah aturan yang diterapkan pada database agar relasi tersebut benar-benar dijaga dengan ketat. Dengan constraint ini, database akan memvalidasi setiap insert/update agar nilai foreign key:

- harus ada di parent table,
- tidak menyebabkan orphaned records,
- dan mengikuti aturan referential integrity.

Ciri-ciri utama constraint:

- Database melakukan pengecekan otomatis.
- Bila datanya tidak valid, operasi query akan gagal.
- Kita bisa menambahkan aksi tambahan seperti `ON DELETE CASCADE`.

Contoh:

```sql
-- Ini adalah foreign key constraint
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id)  -- Dengan REFERENCES, constraint aktif
);
```

Dengan deklarasi `REFERENCES`, database akan memastikan bahwa setiap `state_id` pada tabel `cities` benar-benar ada dalam `states.id`.

#### Kesalahpahaman yang Sering Terjadi

Banyak orang berkata:

> "I don't use foreign keys"
> atau
> "That technology doesn't support foreign keys"

Padahal yang mereka maksud hampir selalu:

> "I don't use foreign key **constraints**"

Karena:

- **Foreign key (konsep)** _pasti ada_ kalau kamu punya relasi antar tabel.
  Mau pakai constraint atau tidak, aplikasi tetap menyimpan ID parent di child table.

- Yang opsional adalah **foreign key constraint**, yaitu apakah database harus ikut menjaga integritas data atau tidak.

Singkatnya:

- Tanpa **foreign key**, relasi antar tabel tidak mungkin dibuat.
- Tanpa **foreign key constraint**, relasi tetap ada, tetapi database tidak memastikan kebenarannya — semua diserahkan ke aplikasi.

Dengan memahami perbedaan ini, kamu bisa lebih sadar kapan sebaiknya menggunakan constraint untuk menjaga integritas, dan kapan kamu memilih untuk mengelolanya di level aplikasi.

### b. Membuat Foreign Key Constraint - Sintaks Dasar

#### Struktur Dasar Parent dan Child Table

Pada bagian ini, kita melihat bagaimana cara membuat foreign key **constraint** secara eksplisit di PostgreSQL. Tujuan utama dari constraint ini adalah memastikan bahwa setiap baris di tabel child selalu memiliki referensi yang valid ke tabel parent.

Pertama, kita membuat tabel _parent_ (`states`) dan tabel _child_ (`cities`). Tabel parent biasanya memiliki primary key yang akan dijadikan acuan oleh tabel child.

```sql
-- Parent table (states)
CREATE TABLE states (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT
);

-- Child table (cities)
CREATE TABLE cities (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id)  -- Foreign key constraint!
);
```

Perhatikan bahwa pada tabel `cities`, kolom `state_id` tidak hanya menjadi kolom biasa yang menyimpan angka, tetapi sudah diberi aturan `REFERENCES states(id)`. Inilah yang mengubahnya dari sekadar foreign key **konsep** menjadi foreign key **constraint**.

#### Syarat-Syarat Penting dalam Foreign Key Constraint

Agar constraint berhasil dibuat dan dapat berjalan dengan benar, ada dua syarat fundamental:

1. **Tipe data harus sama**
   Kolom foreign key (`state_id`) harus memiliki tipe data yang sama dengan kolom yang direferensikan (`states.id`). Jika `states.id` bertipe BIGINT, maka `state_id` juga harus BIGINT.

2. **Kolom yang direferensikan harus UNIQUE**
   Biasanya berupa `PRIMARY KEY` atau kolom yang memiliki `UNIQUE constraint`. Database harus bisa memastikan bahwa nilai yang dirujuk benar-benar unik dan bisa dijadikan acuan.

Tanpa dua aturan ini, constraint referensial tidak dapat dibentuk.

#### Contoh Penggunaan dan Perilaku Constraint

Sekarang mari lihat bagaimana constraint bekerja saat kita melakukan operasi `INSERT`.

```sql
-- Insert parent
INSERT INTO states (name) VALUES ('Texas');
-- Hasil: id = 1
```

Ketika kita menyisipkan data ke tabel `cities`, database akan mengecek apakah nilai `state_id` terdapat di tabel `states`.

```sql
-- ✅ Insert child dengan foreign key valid
INSERT INTO cities (state_id, name) VALUES (1, 'Dallas');
-- Berhasil! state_id = 1 ada di tabel states
```

Karena `state_id = 1` valid (ada di `states.id`), insert berjalan lancar.

Sebaliknya, jika kita mencoba memasukkan nilai yang tidak ada di tabel parent, constraint akan langsung menolak operasi tersebut:

```sql
-- ❌ Insert child dengan foreign key invalid
INSERT INTO cities (state_id, name) VALUES (2, 'Dallas');
-- Error: insert or update on table "cities" violates foreign key constraint
-- Key (state_id)=(2) is not present in table "states"
```

Pesan error dengan jelas mengatakan bahwa nilai `2` tidak ditemukan di tabel `states`, sehingga operasi tidak diizinkan.

#### Ilustrasi Cara Kerja Enforcement

Penegakan constraint ini kurang lebih bisa dibayangkan seperti dialog berikut:

```
User: INSERT INTO cities VALUES (2, 'Dallas')
        ↓
Database: "Tunggu, kamu bilang state_id REFERENCES states(id)"
        ↓
Database: *cek tabel states untuk id = 2*
        ↓
Database: "ID 2 tidak ada! → THROW ERROR"
```

Dengan mekanisme ini, database membantu kita menjaga integritas referensial secara otomatis. Kita tidak perlu menulis kode tambahan di aplikasi untuk memastikan relasi tetap konsisten — semuanya sudah ditangani oleh constraint.

### c. Naming Conventions untuk Foreign Key

#### Pola Penamaan yang Direkomendasikan

Instruktur memberikan pola penamaan yang sederhana, jelas, dan mudah diikuti. Intinya, primary key di tabel parent cukup bernama `id`, sementara foreign key di tabel child menggunakan format **nama_tabel_singular + "\_id"**. Pendekatan ini membuat struktur tabel terlihat ringkas sekaligus tetap mudah dipahami.

```sql
-- Parent table: Kolom ID cukup bernama "id"
CREATE TABLE states (
    id BIGINT PRIMARY KEY,  -- Cukup "id"
    name TEXT
);

-- Child table: Foreign key = singular_table_name + "_id"
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id)  -- "state" (singular) + "_id"
);
```

Pada pola ini, relasinya sangat mudah dibaca: `state_id` di tabel `cities` jelas merupakan foreign key yang menunjuk ke `id` di tabel `states`.

#### Pola Alternatif yang Juga Valid

Selain pola di atas, ada juga pendekatan lain yang cukup umum, yaitu dengan memberi nama primary key berdasarkan nama tabelnya. Pola ini membuat nama kolom lebih eksplisit karena setiap primary key menampilkan identitas tabel secara langsung.

```sql
-- Parent table: Kolom ID pakai nama tabel
CREATE TABLE states (
    state_id BIGINT PRIMARY KEY,  -- "state_id" bukan "id"
    name TEXT
);

-- Child table: Foreign key juga pakai nama yang sama
CREATE TABLE cities (
    city_id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(state_id)  -- Nama sama
);
```

Di pola ini, nama primary key dan foreign key sama persis (`state_id`), sehingga konsistensi penamaan terjaga di setiap tabel yang terhubung.

#### Perbandingan Antara Dua Pola

Untuk membantu memilih pola yang paling sesuai, berikut perbandingan ringkasnya:

| Pattern   | Primary Key | Foreign Key | Kelebihan                    |
| --------- | ----------- | ----------- | ---------------------------- |
| Pattern 1 | `id`        | `state_id`  | Lebih ringkas, jelas mana FK |
| Pattern 2 | `state_id`  | `state_id`  | Konsisten di semua tabel     |

Pattern 1 cenderung terasa lebih “bersih” dan populer di banyak project karena sangat mudah memisahkan mana kolom ID internal tabel dan mana kolom referensi ke tabel lain. Sementara Pattern 2 disukai dalam sistem yang mengutamakan penamaan eksplisit dan konsistensi absolut.

#### Kesimpulan Konvensi Penamaan

Tidak ada pola yang benar atau salah. Keduanya valid secara teknis dan umum digunakan di banyak proyek. Yang paling penting adalah:

- pilih salah satu pola penamaan,
- dan gunakan **secara konsisten** di seluruh aplikasi atau database.

Instruktur sendiri lebih menyukai Pattern 1 (primary key cukup `id`, foreign key menggunakan `table_name_id`) karena struktur tabel terasa lebih minimalis tanpa mengorbankan kejelasan relasi.

### d. Column-Level vs Table-Level Constraint

#### Memahami Dua Cara Menuliskan Foreign Key Constraint

Dalam PostgreSQL (dan SQL secara umum), ada dua cara untuk mendefinisikan foreign key constraint: **column-level** dan **table-level**. Keduanya memiliki fungsi yang sama untuk single-column foreign key, tetapi penempatannya berbeda. Memahami perbedaan ini penting agar struktur tabel tetap rapi dan mudah dikelola.

#### Column-Level Constraint (Inline)

Pada pendekatan column-level, constraint ditulis langsung pada definisi kolom. Ini adalah cara yang paling umum dan paling ringkas untuk foreign key tunggal.

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id)  -- Langsung di definisi kolom
);
```

Dengan cara ini, siapa pun yang membaca definisi kolom akan langsung tahu bahwa `state_id` adalah foreign key. Formatnya jelas dan singkat—cocok untuk kasus yang sederhana.

#### Table-Level Constraint

Pendekatan kedua adalah menuliskan foreign key di bagian akhir definisi tabel, pada blok khusus constraint. Cara ini lebih eksplisit dan memberi fleksibilitas lebih, terutama ketika ada lebih dari satu kolom yang terlibat.

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT,
    FOREIGN KEY (state_id) REFERENCES states(id)  -- Di akhir sebagai table constraint
);
```

Pendekatan ini membuat struktur tabel terlihat lebih terorganisir ketika constraint mulai banyak, misalnya ketika terdapat beberapa foreign key atau constraint lain seperti unique dan check.

#### Kapan Menggunakan Yang Mana?

Untuk membantu memilih secara tepat, berikut pedoman sederhananya:

| Jenis        | Kapan Digunakan                         |
| ------------ | --------------------------------------- |
| Column-level | Single column foreign key (paling umum) |
| Table-level  | Composite foreign key (multi-column)    |

Column-level terasa lebih natural untuk hubungan sederhana satu kolom ke satu parent. Namun, jika foreign key melibatkan beberapa kolom sekaligus (misal `(tenant_id, user_id)`), kolom-kolom tersebut tidak bisa diberi constraint inline; satu-satunya cara adalah menggunakan table-level constraint.

#### Catatan Penting

Kedua cara ini **functionally equivalent** untuk foreign key satu kolom. Database akan memperlakukan keduanya sama. Pemilihan sepenuhnya soal gaya penulisan, konsistensi, dan kenyamanan membaca.

Dengan memahami dua pendekatan ini, kamu bisa memilih gaya yang paling cocok untuk struktur tabelmu—baik yang sederhana maupun yang kompleks.

### e. Composite Foreign Key Constraint

#### Memahami Composite Foreign Key

Composite foreign key digunakan ketika sebuah relasi tidak hanya bergantung pada satu kolom, tetapi pada kombinasi beberapa kolom sekaligus. Ini biasanya muncul ketika tabel parent memiliki **composite unique key**, yaitu kumpulan dua atau lebih kolom yang bersama-sama harus unik.

Mari lihat contoh kasus yang umum: sebuah tabel `states` yang memiliki dua kolom (`abbreviation` dan `code`) yang bersifat unik dalam kombinasinya.

```sql
-- Parent table dengan composite unique
CREATE TABLE states (
    id BIGINT PRIMARY KEY,
    abbreviation CHAR(2),
    code INTEGER,
    name TEXT,
    UNIQUE (abbreviation, code)  -- Composite unique key
);

-- Child table dengan composite foreign key
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_abbreviation CHAR(2),
    state_code INTEGER,
    FOREIGN KEY (state_abbreviation, state_code)
        REFERENCES states(abbreviation, code)
);
```

Pada contoh di atas, tabel `cities` tidak menggunakan `state_id` sebagai foreign key. Sebaliknya, ia menggunakan kombinasi dua kolom: `state_abbreviation` dan `state_code`. Relasi ke tabel `states` hanya valid jika pasangan `(abbreviation, code)` ada dan unik di tabel parent.

#### Aturan Penting dalam Composite Foreign Key

Ada beberapa syarat penting yang perlu dipenuhi agar composite foreign key dapat dibuat dengan benar:

- Kolom-kolom yang direferensikan **harus memiliki UNIQUE constraint**.
  Constraint tersebut bisa berupa `PRIMARY KEY` atau `UNIQUE`, yang penting kombinasi nilai tidak boleh muncul lebih dari sekali.

- Pada composite key, keunikan harus berlaku **bersama-sama**, bukan per kolom.

- Composite foreign key **tidak dapat ditulis sebagai column-level constraint**.
  Karena melibatkan lebih dari satu kolom, definisinya harus berada di table-level.

- Semua kolom child harus memiliki tipe data yang sama dengan kolom parent yang direferensikan.

#### Mengapa Kolom Composite Harus Unik?

Konsepnya sederhana: database harus dapat menelusuri referensi ke satu baris parent secara pasti. Jika kombinasi kolom tidak unik, foreign key menjadi ambigu.

Bayangkan jika tabel states seperti ini:

```
states table:
id | abbreviation
---|-------------
1  | TX
2  | TX  ← Duplikat!
```

Kemudian tabel `cities` ingin mereferensikan `abbreviation = 'TX'`:

- Haruskah ia menunjuk ke baris dengan `id = 1`?
- Atau baris `id = 2`?

Database tidak boleh berada dalam kondisi ambigu seperti ini. Karena itu, kombinasi kolom yang menjadi target foreign key **wajib** unik. Dengan adanya UNIQUE constraint pada `(abbreviation, code)`, database bisa memastikan bahwa setiap pasangan nilai hanya mengarah ke satu baris parent.

Dengan memahami konsep ini, kamu bisa mengelola skema database yang lebih kompleks dan menjaga integritas relasi secara konsisten, meskipun relasi tersebut tidak hanya bergantung pada single-column key.

### f. ON DELETE dan ON UPDATE Actions

#### Memahami Jenis-Jenis Action

Saat membuat foreign key constraint, kita bisa menentukan bagaimana database harus merespons ketika baris di tabel parent dihapus atau di-update. PostgreSQL menyediakan beberapa opsi, masing-masing dengan perilaku berbeda:

| Action        | Deskripsi         | Perilaku                                                           |
| ------------- | ----------------- | ------------------------------------------------------------------ |
| `NO ACTION`   | Default           | Menolak delete/update parent jika masih ada child yang mereferensi |
| `RESTRICT`    | Mirip NO ACTION   | Juga menolak, tetapi pengecekannya tidak bisa ditunda              |
| `CASCADE`     | Cascade operation | Menghapus atau meng-update child mengikuti parent                  |
| `SET NULL`    | Set to NULL       | Nilai foreign key pada child diubah menjadi NULL                   |
| `SET DEFAULT` | Set to default    | Nilai foreign key pada child diubah ke nilai default kolom         |

Setiap action memiliki dampak yang signifikan pada integritas data dan, terutama untuk CASCADE, potensi efek samping yang besar.

---

#### ON DELETE NO ACTION (Default)

Ini adalah perilaku default. Tanpa menulis apa pun, constraint akan menggunakan `NO ACTION`.

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id)
    -- Default: ON DELETE NO ACTION
);

INSERT INTO states (name) VALUES ('Texas');  -- id = 1
INSERT INTO cities (state_id, name) VALUES (1, 'Dallas');

-- Coba delete parent
DELETE FROM states WHERE id = 1;
-- ❌ Error: masih ada cities yang memakai state_id = 1
```

**Intinya:** Parent tidak boleh dihapus jika child masih menggunakannya.

---

#### ON DELETE RESTRICT

Mirip dengan NO ACTION, tetapi pengecekannya dilakukan segera, tanpa menunggu akhir transaksi.

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id) ON DELETE RESTRICT
);

DELETE FROM states WHERE id = 1;
-- ❌ Tetap error
```

##### Perbedaan utama:

- `NO ACTION`: Pengecekan bisa ditunda sampai transaksi selesai.
- `RESTRICT`: Pengecekan langsung saat perintah dijalankan.

Dalam implementasi sehari-hari, efeknya sama: parent tidak dapat dihapus.

---

#### ON DELETE CASCADE (⚠️ HARUS HATI-HATI!)

Dengan CASCADE, jika baris di tabel parent dihapus, semua baris child yang mereferensikannya juga dihapus otomatis.

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id) ON DELETE CASCADE
);

INSERT INTO states (name) VALUES ('Texas');   -- id = 1
INSERT INTO cities (state_id, name) VALUES (1, 'Dallas');

DELETE FROM states WHERE id = 1;
-- Berhasil: baris cities yang terkait ikut terhapus
```

Instruktur memberi peringatan keras:

> "You can imagine a cascading cascade of cascades."

Bayangkan struktur seperti ini:

```
DELETE account
  ↓ CASCADE
  DELETE projects (100 rows)
    ↓ CASCADE
    DELETE tasks (10,000 rows)
      ↓ CASCADE
      DELETE comments (100,000 rows)
        ↓ CASCADE
        DELETE attachments (1,000,000 rows)
```

Satu perintah DELETE dapat menghapus **jutaan baris** tanpa sengaja. Ini membuat audit, debugging, maupun rollback jadi jauh lebih sulit.

##### Rekomendasi:

- Gunakan default (`NO ACTION`) atau `RESTRICT`.
- Hindari CASCADE kecuali kamu benar-benar memahami konsekuensinya.
- Jika butuh efek cascade, lebih aman menghapus secara eksplisit di level aplikasi.

---

#### ON DELETE SET NULL

Action ini akan mengubah nilai foreign key pada child menjadi `NULL` saat parent dihapus.

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id) ON DELETE SET NULL
    -- Catatan: state_id harus NULLABLE
);

INSERT INTO states (name) VALUES ('Texas');  -- id = 1
INSERT INTO cities (state_id, name) VALUES (1, 'Dallas');

DELETE FROM states WHERE id = 1;

SELECT * FROM cities;
-- state_id sekarang NULL
```

##### Kapan berguna?

- Ketika child masih perlu disimpan meskipun parent-nya dihapus.
- Ketika foreign key bersifat opsional dan tidak harus selalu terhubung.
- Untuk kasus di mana orphaned rows tidak masalah dan bisa di-cleanup belakangan.

Syarat penting: kolom harus boleh `NULL`.

---

#### ON DELETE SET DEFAULT

Action ini mengatur nilai foreign key pada child menjadi nilai default kolom.

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT DEFAULT 1 REFERENCES states(id) ON DELETE SET DEFAULT
);
```

Kalau baris parent dengan `id = 1` dihapus, database akan mencoba mengisi `state_id` dengan nilai default (`1`). Tetapi ini hanya valid jika nilai itu memang masih ada di tabel parent—jika tidak, akan terjadi error.

##### Kapan digunakan?

- Jarang dipakai.
- Biasanya hanya untuk model yang punya nilai default khusus seperti "unknown" atau "unassigned".

### g. ON UPDATE Actions

Pada bagian ini, kita membahas bagaimana sebuah foreign key merespons perubahan pada **nilai primary key** di tabel parent. Secara konsep, mekanismenya sama seperti _ON DELETE_, hanya saja fokusnya pada operasi **UPDATE**, bukan **DELETE**.

#### Struktur Dasar Penggunaan ON UPDATE

Sintaks penulisannya identik dengan ON DELETE. Kamu cukup menambahkannya setelah deklarasi foreign key:

```sql
CREATE TABLE cities (
    id BIGINT PRIMARY KEY,
    name TEXT,
    state_id BIGINT REFERENCES states(id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT
);
```

Di sini, kolom `state_id` menjadi foreign key yang mengarah ke `states.id`. Dua aturan diterapkan:

- `ON UPDATE CASCADE` → Jika `states.id` berubah, maka semua `state_id` di tabel `cities` ikut berubah mengikuti nilai barunya.
- `ON DELETE RESTRICT` → Tidak boleh menghapus row di `states` selama masih direferensikan oleh `cities`.

#### Perilaku ON UPDATE

Agar lebih mudah dipahami, mari kita uraikan efek tiap opsi ON UPDATE.

##### ON UPDATE CASCADE

Aturan ini memberi tahu database bahwa jika primary key di parent berubah, database harus secara otomatis memperbarui nilai foreign key di tabel child agar tetap konsisten.

Contoh skenario:

```sql
INSERT INTO states (id, name) VALUES (1, 'Texas');
INSERT INTO cities (name, state_id) VALUES ('Dallas', 1);

-- Parent di-update
UPDATE states SET id = 2 WHERE id = 1;
```

Jika kamu menggunakan `ON UPDATE CASCADE`, maka baris pada `cities` otomatis berubah menjadi:

```
id | name   | state_id
---|--------|---------
1  | Dallas | 2
```

Dengan kata lain, database menjaga referensi tetap valid tanpa campur tangan dari aplikasi.

##### ON UPDATE RESTRICT

Kebalikan dari CASCADE, opsi ini mencegah perubahan primary key parent jika masih ada baris yang mereferensikannya.

Contoh:

```sql
UPDATE states SET id = 2 WHERE id = 1;
```

Dengan `ON UPDATE RESTRICT`, hasilnya:

```
❌ ERROR: update on table "states" violates foreign key constraint
```

Database akan menolak perubahan tersebut untuk mencegah terjadinya foreign key yang "patah".

#### Catatan Penting dalam Praktik

Walaupun ON UPDATE bisa sangat powerful, dalam praktik sehari-hari fitur ini jarang sekali digunakan. Alasannya sederhana:

- Primary key _hampir tidak pernah_ berubah.
- Pada desain modern, primary key biasanya adalah:

  - `SERIAL` atau `GENERATED AS IDENTITY`
  - atau `UUID`

Keduanya bersifat stabil — begitu dibuat, nilainya tidak berubah. Karena itu, kebutuhan untuk melakukan cascade update sangat kecil.

Dengan kata lain, ON UPDATE lebih bersifat opsional. Ia tetap berguna untuk kasus tertentu (misalnya jika kamu memakai natural key yang memungkinkan perubahan), tetapi dalam desain schema database yang umum, opsi ini jarang diperlukan.

### h. Contoh Lengkap dengan Best Practices

Pada bagian ini, kita melihat contoh lengkap bagaimana mendefinisikan relasi antara tabel parent (`states`) dan tabel child (`cities`) dengan foreign key yang dirancang mengikuti praktik terbaik (_best practices_). Contoh ini bukan hanya menunjukkan sintaksnya, tetapi juga alasan di balik setiap pilihan desain.

#### Contoh Schema dengan Foreign Key yang Baik

Pertama, kita buat tabel parent `states`:

```sql
-- Parent table
CREATE TABLE states (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    abbreviation CHAR(2) NOT NULL UNIQUE
);
```

Beberapa hal penting yang dilakukan di sini:

- `id` menggunakan `GENERATED ALWAYS AS IDENTITY`, sehingga nilai primary key selalu unik dan tidak bisa diubah manual.
- Kolom `name` dan `abbreviation` bersifat `NOT NULL`, memastikan data penting tidak kosong.
- `abbreviation` dibuat `UNIQUE` agar tidak ada dua state memakai kode yang sama.

Selanjutnya, kita definisikan tabel `cities` yang memiliki foreign key menuju `states.id`:

```sql
-- Child table dengan foreign key constraint
CREATE TABLE cities (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    population INTEGER,
    state_id BIGINT NOT NULL REFERENCES states(id)
        ON DELETE RESTRICT      -- Tidak bisa delete state jika ada cities
        ON UPDATE CASCADE,      -- Update state.id → cities.state_id ikut update
    CONSTRAINT cities_state_fk
        FOREIGN KEY (state_id) REFERENCES states(id)  -- Named constraint
);
```

Di sini ada beberapa aspek penting yang layak diperhatikan:

- `state_id` didefinisikan sebagai `NOT NULL`, artinya setiap city _harus_ berada dalam sebuah state. Jika secara desain city tidak boleh berdiri sendiri, maka aturan ini sangat tepat.
- `ON DELETE RESTRICT` mencegah kita menghapus sebuah row di `states` sebelum semua `cities` yang mereferensikannya dihapus atau dipindahkan terlebih dahulu. Ini menjaga integritas data dan menghindari deletion chain yang tidak disengaja.
- `ON UPDATE CASCADE` memastikan bahwa jika `states.id` berubah (meskipun jarang terjadi), nilai `cities.state_id` akan ikut berubah otomatis. Hal ini menjaga referensi tetap konsisten.
- Constraint diberi nama (`cities_state_fk`), sehingga jika terjadi error, pesan yang muncul lebih mudah dipahami dan proses debugging menjadi lebih cepat.

#### Best Practices yang Diterapkan

Beberapa keputusan desain dalam schema di atas mengikuti praktik terbaik dalam membuat foreign key:

#### 1. Menandai foreign key sebagai NOT NULL (jika memang wajib ada)

Jika secara logika sebuah city harus selalu berada dalam satu state, maka `state_id` tidak boleh kosong. Ini menghindari data tidak lengkap yang bisa membingungkan aplikasi atau laporan.

#### 2. Menggunakan ON DELETE RESTRICT

Ini adalah pilihan default yang aman. Kita tidak ingin satu perintah `DELETE` pada tabel `states` tanpa sengaja menghapus seluruh jaringan data terkait di `cities`.

#### 3. ON UPDATE CASCADE untuk menjaga konsistensi

Walaupun primary key jarang berubah, menggunakan ON UPDATE CASCADE tidak berbahaya. Jika suatu saat ID berubah (misalnya migrasi data), child table tetap konsisten otomatis.

#### 4. Memberikan nama pada foreign key constraint

Menamai constraint seperti `cities_state_fk` sangat membantu saat terjadi error. Pesan error akan menunjukkan nama constraint tersebut sehingga kita langsung tahu relasi mana yang bermasalah.

Dengan kombinasi praktik terbaik ini, schema menjadi lebih mudah dipahami, lebih aman dari operasi destruktif yang tidak disengaja, dan lebih jelas saat terjadi error.

## 3. Hubungan Antar Konsep

### Alur Pemahaman:

```
1. Konsep Relational Database
   ↓
   Tables saling berhubungan via ID

2. Foreign Key (Konsep)
   ↓
   Kolom yang berisi ID dari tabel lain
   ↓
   Ini SELALU ada dalam relational design

3. Foreign Key Constraint (Enforcement)
   ↓
   OPTIONAL: Apakah mau enforce referential integrity?
   ↓
   Jika ya → tambahkan REFERENCES clause

4. Enforcement Behavior
   ├─ ON DELETE: Apa yang terjadi saat parent dihapus?
   │  ├─ NO ACTION/RESTRICT: Cegah delete parent
   │  ├─ CASCADE: Delete child juga (BAHAYA!)
   │  ├─ SET NULL: Set foreign key child jadi NULL
   │  └─ SET DEFAULT: Set foreign key child ke default
   │
   └─ ON UPDATE: Apa yang terjadi saat parent ID berubah?
      └─ (Jarang digunakan, biasanya CASCADE atau RESTRICT)
```

### Diagram Referential Integrity:

```
WITHOUT Foreign Key Constraint:
================================
states                cities
id | name              id | name   | state_id
---|------             ---|--------|----------
1  | Texas             1  | Dallas | 1
                       2  | Austin | 99  ← Orphaned! (state_id 99 tidak ada)

Database: "Ok, I'll insert it" ✅ (No validation)


WITH Foreign Key Constraint:
============================
states                cities (state_id REFERENCES states(id))
id | name              id | name   | state_id
---|------             ---|--------|----------
1  | Texas             1  | Dallas | 1

INSERT cities VALUES (2, 'Austin', 99)
                       ↓
Database: "Wait, let me check states table..."
                       ↓
Database: "ID 99 not found → REJECT" ❌
```

### Decision Tree: Memilih ON DELETE Action

```
Apakah child rows masih relevan tanpa parent?
├─ YA (contoh: archived data, historical records)
│  └─ Gunakan: SET NULL atau SET DEFAULT
│     └─ Cleanup orphaned rows secara manual/batch job
│
└─ TIDAK (child tidak ada artinya tanpa parent)
   ├─ Apakah delete parent sering terjadi?
   │  ├─ YA → CASCADE mungkin OK (tapi hati-hati!)
   │  │  └─ ⚠️ Pastikan tidak ada cascade chain yang panjang
   │  │
   │  └─ TIDAK → RESTRICT/NO ACTION (safest!)
   │     └─ Force application to explicitly delete children first
   │
   └─ Default choice: RESTRICT atau NO ACTION
```

### Hierarchy of Safety:

```
PALING AMAN (Recommended)
↓
1. ON DELETE RESTRICT / NO ACTION
   - Tidak ada surprise deletes
   - Explicit handling di application

2. ON DELETE SET NULL
   - Orphaned rows tapi masih ada
   - Bisa cleanup nanti

3. ON DELETE CASCADE
   - Convenient tapi DANGEROUS
   - Bisa delete millions of rows accidentally
↓
PALING BERBAHAYA
```

### Hubungan dengan Konsep Lain:

**Foreign Key + Indexing (akan dibahas di modul joins):**

```sql
-- Foreign key constraint ≠ Index
CREATE TABLE cities (
    state_id BIGINT REFERENCES states(id)  -- Ada constraint
);

-- Tapi belum ada index!
-- Untuk performa joins, perlu add index manually:
CREATE INDEX idx_cities_state_id ON cities(state_id);
```

**Foreign Key + UNIQUE Constraint:**

```sql
-- Yang di-reference HARUS UNIQUE
CREATE TABLE states (
    id BIGINT PRIMARY KEY,        -- PRIMARY KEY = UNIQUE
    abbreviation CHAR(2) UNIQUE   -- Explicit UNIQUE
);

-- Bisa reference ke kolom manapun yang UNIQUE
CREATE TABLE cities (
    state_id BIGINT REFERENCES states(id),           -- ✅ OK
    state_abbr CHAR(2) REFERENCES states(abbreviation) -- ✅ OK juga
);
```

## 4. Kesimpulan

**Key Takeaways:**

1. **Foreign Key ≠ Foreign Key Constraint:**

   - Foreign Key = Konsep relasional (selalu ada)
   - Foreign Key Constraint = Enforcement (optional, tapi recommended)

2. **Membuat Foreign Key Constraint:**

   ```sql
   column_name TYPE REFERENCES parent_table(column)
   ```

3. **Syarat mutlak:**

   - Data types harus match
   - Kolom yang di-reference harus UNIQUE (PRIMARY KEY atau UNIQUE constraint)

4. **Naming conventions:**

   - Pattern 1: `id` di parent, `table_name_id` di child (recommended)
   - Pattern 2: `table_name_id` di semua tabel
   - Yang penting: **konsisten!**

5. **Column-level vs Table-level:**

   - Column-level: Untuk single column FK
   - Table-level: Untuk composite FK (wajib)

6. **ON DELETE Actions:**

   - `NO ACTION` / `RESTRICT`: Default, safest (recommended)
   - `CASCADE`: Convenient tapi dangerous (hati-hati!)
   - `SET NULL`: Orphaned rows untuk cleanup nanti
   - `SET DEFAULT`: Jarang digunakan

7. **Best Practice dari instruktur:**
   - ✅ Gunakan foreign key constraints untuk referential integrity
   - ✅ Prefer ON DELETE RESTRICT atau NO ACTION
   - ⚠️ Hindari ON DELETE CASCADE (atau gunakan dengan sangat hati-hati)
   - ✅ Jika perlu cascade, handle di application level dengan explicit deletes

**Manfaat Foreign Key Constraint:**

- Menjamin referential integrity
- Mencegah orphaned rows
- Database-level enforcement (tidak perlu handle di application)
- Self-documenting relationships

**Yang akan dibahas nanti:**

- Indexing foreign keys untuk performa joins (modul joins)
- Composite foreign keys detail
- Advanced scenarios dengan multi-table relationships

**Prinsip emas:** "Foreign key constraints give me referential integrity, but be careful with CASCADE - it can explode into millions of deletes!"
