# 📝 Partial Indexes (Indeks Parsial) di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas fitur **partial index** (indeks parsial) yang tersedia di PostgreSQL dan SQLite (namun **tidak** ada di MySQL). Partial index memungkinkan kita membuat indeks hanya untuk sebagian baris dalam tabel — yakni baris yang memenuhi suatu kondisi (predicate) tertentu. Ini sangat berguna ketika data kita berdistribusi tidak merata (_highly skewed_), di mana hanya sebagian kecil baris yang sering di-query. Selain untuk performa query, partial index juga bisa digunakan untuk menegakkan **partial unique constraint** — misalnya untuk mendukung fitur soft delete.

---

## 2. Konsep Utama

### 📝 Partial Indexes (Indeks Parsial) di PostgreSQL

#### Ringkasan Singkat

Partial index adalah fitur di PostgreSQL (dan juga SQLite) yang memungkinkan kita membuat indeks **hanya untuk sebagian data** dalam sebuah tabel—tepatnya, hanya baris yang memenuhi kondisi tertentu (_predicate_).

Ini sangat berguna ketika distribusi data tidak merata (_highly skewed_). Misalnya, dari jutaan data, hanya sebagian kecil yang sering diakses. Daripada mengindeks semuanya, kita cukup mengindeks bagian yang relevan saja.

Selain meningkatkan performa query, partial index juga bisa dipakai untuk membuat **aturan keunikan bersyarat (partial unique constraint)**—contohnya untuk mendukung fitur soft delete.

---

#### Konsep Utama

##### Apa Itu Partial Index?

Secara sederhana, partial index adalah indeks yang hanya berisi **subset data** dari sebuah tabel.

Berbeda dengan indeks biasa yang mencakup semua baris, partial index hanya menyimpan baris yang lolos dari kondisi tertentu.

Contohnya:

```sql
-- Index biasa (semua baris)
CREATE INDEX idx_email ON users (email);

-- Partial index (hanya user pro)
CREATE INDEX idx_email_pro ON users (email) WHERE is_pro = true;
```

Pada contoh kedua, struktur B-tree hanya berisi email dari user dengan `is_pro = true`. Data user non-pro sama sekali tidak masuk ke indeks.

Perlu dicatat:

- Ini **bukan composite index**, karena tetap hanya satu kolom.
- Perbedaannya ada pada **filter baris**, bukan jumlah kolom.

---

##### Cara Kerja Partial Index

Hal penting yang sering terlewat: **query harus “match” dengan kondisi indeks** agar bisa digunakan.

PostgreSQL tidak akan “menebak” bahwa partial index relevan jika kondisi tersebut tidak disebutkan di query.

Contoh yang **tidak menggunakan partial index**:

```sql
SELECT * FROM users
WHERE email = 'aaron.francis@example.com';
```

Kenapa tidak dipakai? Karena tidak ada kondisi `is_pro = true`, sehingga planner tidak yakin indeks tersebut relevan.

Contoh yang **menggunakan partial index**:

```sql
SELECT * FROM users
WHERE email = 'aaron.francis@example.com'
  AND is_pro = true;
```

Di sini, kondisi query cocok dengan predicate indeks → PostgreSQL bisa menggunakan index scan.

Poin penting:

- Partial index hanya berlaku untuk kondisi yang sesuai.
- Query dengan `is_pro = false` atau tanpa kondisi tersebut akan mengabaikan indeks ini.

---

##### Keuntungan Partial Index

Dengan hanya menyimpan data yang relevan, kita mendapatkan beberapa keuntungan nyata:

**1. Query SELECT lebih cepat**
Karena ukuran indeks lebih kecil, traversal B-tree jadi lebih efisien.

**2. Operasi write lebih ringan**
INSERT, UPDATE, dan DELETE tidak perlu memperbarui indeks untuk baris yang tidak termasuk dalam predicate.

**3. Ukuran indeks lebih kecil**
Tidak ada “bloat” dari data yang sebenarnya tidak kita butuhkan.

Bayangkan kasus seperti ini:

```
Total data: 1.000.000 baris
- User pro:        20.000  → masuk indeks
- User non-pro:   980.000  → diabaikan
```

Partial index hanya menyimpan 20.000 baris, bukan 1 juta. Dampaknya terasa langsung pada performa.

---

##### Partial Unique Index

Selain untuk performa, partial index juga bisa digabung dengan constraint `UNIQUE`.

Tujuannya: memastikan keunikan **hanya pada subset tertentu**.

Contoh paling umum adalah fitur soft delete.

Kebutuhannya:

- Email harus unik untuk user aktif
- User yang sudah dihapus boleh punya email yang sama

Solusinya:

```sql
CREATE UNIQUE INDEX idx_unique_active_email
ON users (email)
WHERE deleted_at IS NULL;
```

Artinya:

- Selama `deleted_at IS NULL` → email harus unik
- Jika `deleted_at IS NOT NULL` → boleh duplikat

Contoh skenario:

```sql
-- Ada dua user dengan email sama:
-- Aaron (aktif)
-- Cleo  (sudah dihapus)

-- Tidak masalah saat membuat index
-- karena hanya Aaron yang aktif

-- Tapi jika Cleo diaktifkan kembali:
UPDATE users SET deleted_at = NULL WHERE name = 'Cleo Simonis';
```

Hasilnya:

```
ERROR: duplicate key value violates unique constraint
```

Karena sekarang ada dua user aktif dengan email yang sama.

---

##### Kasus Penggunaan Lain

Partial unique index bisa dipakai untuk berbagai aturan bisnis yang lebih kompleks, misalnya:

**1. Status order**

```sql
CREATE UNIQUE INDEX idx_unique_confirmed_order_ref
ON orders (reference_number)
WHERE status != 'draft';
```

Hanya order non-draft yang harus unik.

**2. Artikel (draft vs published)**
Slug hanya harus unik untuk artikel yang sudah dipublish.

**3. Booking sistem**
Hanya booking yang sudah dikonfirmasi yang harus unik dalam satu slot waktu.

Intinya, partial index memungkinkan kita memindahkan sebagian logic bisnis ke level database.

Tapi hati-hati:
Definisi predicate harus benar-benar sesuai dengan kebutuhan bisnis. Salah sedikit saja bisa membuat constraint terlalu longgar atau terlalu ketat.

---

#### Hubungan Antar Konsep

Partial index sangat berkaitan dengan dua konsep penting dalam database:

- **Cardinality** → jumlah nilai unik dalam kolom
- **Selectivity** → seberapa efektif filter dalam menyaring data

Ketika data sangat tidak merata (misalnya hanya 2% yang relevan), membuat indeks penuh sering kali tidak efisien.

Di sinilah partial index jadi solusi:

- Fokus hanya pada subset dengan selectivity tinggi
- Menghindari overhead dari data yang jarang dipakai

Alurnya bisa dipahami seperti ini:

```
Data skewed (tidak merata)
        ↓
Subset tertentu sering di-query
        ↓
Gunakan Partial Index
        ↓
- Untuk performa → index biasa + WHERE
- Untuk constraint → UNIQUE + WHERE
```

---

#### Kesimpulan

Partial index adalah fitur yang sangat powerful untuk mengoptimalkan performa sekaligus menegakkan aturan bisnis yang fleksibel.

Manfaat utamanya:

- Query lebih cepat karena indeks lebih kecil
- Operasi write lebih ringan
- Bisa membuat constraint bersyarat (partial unique)

Hal yang perlu diingat:

1. Query harus menyertakan kondisi yang sesuai dengan predicate indeks.
2. Jangan asal membuat partial unique index—pahami dulu business logic-nya.
3. Partial index bukan composite index—ini tetap indeks satu kolom dengan filter baris.

Kalau digunakan dengan tepat, partial index bisa jadi salah satu optimasi paling impactful dalam desain database.

#### Cara Kerja Partial Index (Planner Harus Tahu Predicate-nya)

Salah satu hal paling penting dalam memahami partial index adalah: **PostgreSQL tidak akan menggunakan indeks ini secara otomatis kecuali query kita “selaras” dengan kondisi (predicate) yang digunakan saat membuat indeks.**

Artinya, _query planner_ di PostgreSQL hanya akan mempertimbangkan partial index jika ia bisa memastikan bahwa kondisi dalam query **pasti memenuhi predicate indeks tersebut**.

Kenapa begitu? Karena partial index hanya berisi sebagian data. Jika query tidak menyebutkan kondisi yang sama, PostgreSQL tidak punya jaminan bahwa data yang dicari memang ada di dalam indeks tersebut. Akibatnya, ia akan memilih cara yang lebih aman, yaitu melakukan _sequential scan_ (membaca seluruh tabel).

---

#### Contoh: Query Tidak Menggunakan Partial Index

```sql
-- Predicate is_pro = true tidak disebutkan → sequential scan
SELECT * FROM users WHERE email = 'aaron.francis@example.com';
```

Pada query ini, kita hanya mencari berdasarkan `email`. Tidak ada informasi apakah user tersebut `is_pro = true` atau tidak.

Dari sudut pandang PostgreSQL:

- Partial index hanya berisi data dengan `is_pro = true`
- Tapi query tidak menyebutkan kondisi tersebut
- Jadi, PostgreSQL tidak bisa memastikan apakah baris yang dicari ada di dalam indeks

Akibatnya, query planner memilih **tidak menggunakan indeks** dan melakukan _sequential scan_.

---

#### Contoh: Query Menggunakan Partial Index

```sql
-- Predicate cocok → index scan dengan partial index
SELECT * FROM users
WHERE email = 'aaron.francis@example.com'
  AND is_pro = true;
```

Di sini, kondisi `is_pro = true` ditambahkan secara eksplisit.

Sekarang PostgreSQL bisa menyimpulkan:

- Semua data dalam partial index memenuhi `is_pro = true`
- Query juga meminta data dengan kondisi tersebut
- Artinya, indeks ini **relevan dan aman untuk digunakan**

Hasilnya, PostgreSQL akan menggunakan **index scan** yang jauh lebih efisien dibanding membaca seluruh tabel.

---

#### Kenapa Harus “Match” Secara Eksplisit?

Partial index tidak seperti indeks biasa yang mencakup semua data. Karena itu, PostgreSQL tidak bisa “berasumsi” atau menebak kondisi yang tidak tertulis.

Bahkan jika secara logika aplikasi kita tahu bahwa user tersebut pasti pro, PostgreSQL tetap tidak akan menggunakan indeks jika kondisi itu tidak ada di query.

---

#### Batasan Penting: Partial Index Bersifat Satu Arah

Partial index hanya berlaku untuk kondisi yang sesuai dengan predicate-nya.

Artinya:

- Query dengan `is_pro = true` → bisa menggunakan indeks
- Query dengan `is_pro = false` → tidak bisa
- Query tanpa kondisi `is_pro` → juga tidak bisa

Jadi, penting untuk selalu memastikan bahwa query yang ingin dioptimasi memang menyertakan kondisi yang sesuai dengan desain partial index yang dibuat.

---

#### Intuisi Sederhana

Bayangkan partial index seperti rak khusus di perpustakaan yang hanya berisi buku kategori tertentu.

Jika kamu datang dan bilang:

> “Saya cari buku dengan judul X”

Petugas tidak akan langsung ke rak khusus, karena belum tentu buku itu ada di sana.

Tapi jika kamu bilang:

> “Saya cari buku kategori Pro dengan judul X”

Baru masuk akal untuk langsung mencari di rak khusus tersebut.

Itulah cara PostgreSQL “berpikir” saat memutuskan apakah partial index bisa digunakan atau tidak.

#### Keuntungan Partial Index

Ketika kita menggunakan partial index, kita sebenarnya sedang membuat struktur indeks (B-tree) yang **hanya berisi data yang benar-benar relevan** dengan kebutuhan query kita.

Alih-alih menyimpan seluruh baris dalam tabel, kita hanya menyimpan subset tertentu. Dampaknya tidak hanya ke performa query, tapi juga ke efisiensi sistem secara keseluruhan.

Mari kita bahas satu per satu keuntungannya.

---

#### Query SELECT Lebih Cepat

Karena partial index hanya berisi sebagian data, ukuran B-tree menjadi jauh lebih kecil.

Semakin kecil B-tree:

- Semakin sedikit node yang perlu dilalui
- Semakin cepat proses pencarian (traversal)

Artinya, ketika query menggunakan indeks ini, PostgreSQL bisa menemukan data dengan lebih cepat dibandingkan harus menelusuri indeks penuh yang besar.

Ini sangat terasa pada tabel besar dengan jutaan baris, terutama jika hanya sebagian kecil data yang sering di-query.

---

#### Operasi INSERT, UPDATE, DELETE Lebih Ringan

Setiap kali kita melakukan operasi write (INSERT, UPDATE, DELETE), PostgreSQL biasanya harus:

- Memperbarui tabel
- Sekaligus memperbarui semua indeks yang terkait

Pada full index, **semua baris terindeks**, jadi setiap perubahan akan memicu update ke struktur indeks.

Namun pada partial index:

- Hanya baris yang memenuhi predicate yang masuk ke indeks
- Baris lain diabaikan

Artinya:

- Jika sebuah baris tidak termasuk dalam predicate, PostgreSQL **tidak perlu menyentuh indeks sama sekali**

Bayangkan situasi ini:

- Total data: 1.000.000 baris
- Yang relevan hanya 20.000 baris
- Sisanya 980.000 baris sebenarnya tidak pernah digunakan dalam query tertentu

Tanpa partial index, semua perubahan pada 1 juta baris itu tetap akan memengaruhi indeks.
Dengan partial index, hanya 20.000 baris yang “membebani” indeks.

Ini membuat operasi write jadi jauh lebih ringan dan efisien.

---

#### Ukuran Indeks Lebih Kecil

Karena hanya menyimpan subset data, ukuran indeks otomatis lebih kecil.

Dampaknya:

- Menghemat storage
- Mengurangi kemungkinan _bloat_ (pembengkakan indeks)
- Lebih ramah terhadap cache (lebih banyak bagian indeks bisa masuk ke memory)

Semua ini berkontribusi langsung ke performa yang lebih stabil dan cepat.

---

#### Ilustrasi Sederhana

```id="p3m6k9"
Tabel users: 1.000.000 baris
├── Pengguna pro (is_pro = true):  ~20.000 baris  → masuk partial index
└── Pengguna biasa:               ~980.000 baris  → tidak masuk
```

Dengan pendekatan ini:

- Partial index hanya berisi 20.000 entri
- Full index harus menyimpan 1.000.000 entri

Perbedaannya sangat signifikan, baik dari sisi ukuran maupun performa.

---

#### Intuisi Praktis

Cara mudah memahaminya:
Partial index itu seperti kamu hanya menyimpan “shortcut” ke data yang sering kamu pakai, bukan ke semua data.

Hasilnya:

- Pencarian lebih cepat
- Update lebih hemat kerja
- Struktur data lebih ringan

Itulah alasan kenapa partial index sering jadi optimasi yang sangat efektif, terutama pada data yang distribusinya tidak merata.

### d. Partial Unique Index

Partial index tidak hanya berguna untuk meningkatkan performa query, tapi juga bisa digunakan untuk menegakkan aturan data yang lebih fleksibel melalui kombinasi dengan constraint `UNIQUE`.

Ketika digabungkan dengan `UNIQUE`, kita tidak lagi membatasi keunikan untuk seluruh tabel, melainkan hanya untuk **subset data tertentu** yang kita definisikan lewat predicate.

Dengan kata lain, kita bisa mengatakan:

> “Data harus unik, tapi hanya dalam kondisi tertentu.”

---

#### Kasus Nyata: Soft Delete

Salah satu use case paling umum untuk partial unique index adalah fitur **soft delete**.

Dalam soft delete:

- Data tidak benar-benar dihapus dari database
- Tapi hanya ditandai sebagai “dihapus” menggunakan kolom seperti `deleted_at`

Contoh kebutuhan bisnisnya:

- Email harus unik di antara user yang **masih aktif**
- User yang sudah dihapus boleh menggunakan email yang sama lagi

Misalnya:

- Aaron (aktif) → email: [aaron@example.com](mailto:aaron@example.com)
- Cleo (sudah dihapus) → email: [aaron@example.com](mailto:aaron@example.com)

Secara bisnis, ini boleh terjadi karena Cleo sudah tidak dianggap aktif lagi.

Namun jika kita menggunakan `UNIQUE INDEX` biasa pada kolom `email`, PostgreSQL akan langsung menolak kondisi ini, karena ia melihat duplikasi tanpa memperhatikan status aktif atau tidaknya user.

---

#### Solusi: Partial Unique Index

Untuk menyelesaikan masalah ini, kita bisa membatasi aturan unique hanya pada user yang masih aktif:

```sql id="v8q1e7"
-- Hanya user aktif yang wajib memiliki email unik
CREATE UNIQUE INDEX idx_unique_active_email
ON users (email)
WHERE deleted_at IS NULL;
```

Dengan definisi ini, kita mengubah cara kerja constraint menjadi lebih spesifik:

- Jika `deleted_at IS NULL` (user aktif) → email harus unik
- Jika `deleted_at IS NOT NULL` (user sudah dihapus) → email boleh duplikat

Jadi PostgreSQL tidak lagi melihat seluruh tabel, tetapi hanya subset yang relevan.

---

#### Dampak Perilaku di Sistem

Mari kita lihat bagaimana aturan ini bekerja dalam praktik.

Misalnya kondisi awal:

- Aaron Francis → aktif → `deleted_at = NULL`
- Cleo Simonis → sudah dihapus → `deleted_at = '2024-01-01'`

Keduanya memiliki email yang sama, tapi tidak masalah karena:

- Hanya Aaron yang masuk kategori “aktif”
- Cleo diabaikan oleh partial unique index

Sehingga pembuatan index berhasil tanpa error.

---

#### Verifikasi Constraint

Sekarang kita lihat kasus ketika constraint dilanggar secara tidak langsung:

```sql id="z5t8ka"
-- Mengaktifkan kembali user yang sudah dihapus
UPDATE users
SET deleted_at = NULL
WHERE name = 'Cleo Simonis';
```

Saat query ini dijalankan, PostgreSQL akan memeriksa kembali constraint unique pada subset `deleted_at IS NULL`.

Sekarang kondisi berubah menjadi:

- Aaron → aktif → email: [aaron@example.com](mailto:aaron@example.com)
- Cleo → aktif lagi → email: [aaron@example.com](mailto:aaron@example.com)

Artinya:

- Ada dua user aktif dengan email yang sama
- Ini melanggar partial unique index

Hasilnya:

```text
ERROR: duplicate key value violates unique constraint
```

---

#### Intuisi Penting

Partial unique index memungkinkan kita untuk “memindahkan logika bisnis” ke level database.

Alih-alih menulis logic seperti ini di aplikasi:

- Cek apakah email unik hanya di user aktif
- Abaikan user yang sudah dihapus

Kita cukup mendefinisikannya sekali di database:

- Unique hanya berlaku untuk `deleted_at IS NULL`

Ini membuat:

- Data lebih konsisten
- Logic lebih aman (karena enforced di database, bukan hanya di aplikasi)
- Risiko bug karena lupa validasi di layer aplikasi menjadi lebih kecil

---

#### Kesimpulan Konsep

Partial unique index pada dasarnya adalah cara untuk mengatakan:

> “Keunikan data tidak selalu berlaku di seluruh tabel, tapi hanya pada kondisi tertentu.”

Dengan pendekatan ini, kita bisa membuat aturan yang:

- Lebih fleksibel
- Lebih dekat dengan real business logic
- Tetap aman karena dijaga langsung oleh database

Ini sangat berguna pada sistem yang memiliki status data seperti:

- Active vs deleted
- Draft vs published
- Pending vs confirmed

dan berbagai skenario lain yang berbasis status.

### e. Kasus Penggunaan Lain: Business Logic yang Kompleks

Partial unique index sebenarnya tidak terbatas hanya pada kasus soft delete saja. Lebih dari itu, fitur ini sangat berguna ketika kita ingin menerapkan aturan bisnis yang bergantung pada **status atau kondisi tertentu dari data**.

Intinya, kita bisa membuat aturan:

> “Data harus unik, tapi hanya ketika berada dalam kondisi tertentu.”

Pendekatan ini sangat kuat karena memungkinkan database ikut “memahami” business logic aplikasi kita.

---

#### Contoh Kasus Business Logic

Ada banyak skenario di dunia nyata yang cocok menggunakan partial unique index:

##### 1. Status Order

Dalam sistem e-commerce, biasanya sebuah order bisa memiliki beberapa status, seperti:

- draft
- active
- completed
- cancelled

Kebutuhan bisnisnya:

- Nomor referensi order harus unik selama order masih aktif
- Tapi kalau order sudah selesai atau dibatalkan, nomor tersebut boleh dipakai lagi

Dengan kata lain, kita hanya peduli ke order yang “aktif”.

---

##### 2. Draft vs Published (Artikel)

Pada sistem CMS (Content Management System):

- Artikel yang masih draft tidak terlalu penting untuk aturan keunikan
- Tapi artikel yang sudah dipublish harus memiliki slug yang unik

Artinya:

- `draft` boleh punya slug yang sama
- `published` harus unik

Ini sangat umum pada sistem blog atau platform konten.

---

##### 3. Confirmed Bookings

Dalam sistem booking (misalnya hotel atau reservasi):

- Booking yang sudah dikonfirmasi harus unik dalam satu slot waktu
- Tapi booking yang masih pending atau dibatalkan tidak perlu ikut aturan ini

Jadi constraint hanya berlaku ketika status booking sudah “confirmed”.

---

#### Contoh Implementasi SQL

Mari kita lihat bagaimana partial unique index diterapkan pada kasus order:

```sql id="m7q9ld"
-- Unique hanya untuk order yang bukan draft
CREATE UNIQUE INDEX idx_unique_confirmed_order_ref
ON orders (reference_number)
WHERE status != 'draft';
```

---

#### Penjelasan Cara Kerja

Dengan definisi ini, PostgreSQL akan menerapkan aturan berikut:

- Jika `status = 'draft'`
  → Tidak ada constraint unique, data boleh duplikat

- Jika `status != 'draft'` (misalnya active, completed, cancelled)
  → `reference_number` harus unik

Artinya, database secara otomatis membedakan mana data yang “penting untuk dijaga keunikannya” dan mana yang tidak.

---

#### Kenapa Ini Powerful?

Tanpa partial unique index, kita biasanya harus:

- Menulis logic validasi di aplikasi
- Menambahkan banyak kondisi di query
- Berpotensi lupa menangani edge case tertentu

Dengan partial unique index:

- Rule ditulis sekali di database
- Konsistensi dijaga otomatis oleh PostgreSQL
- Risiko bug karena logic yang tidak sinkron jadi lebih kecil

---

#### Catatan Penting

⚠️ Partial unique index adalah fitur yang sangat powerful, tapi juga sensitif terhadap desain bisnis.

Kesalahan dalam menentukan predicate bisa berdampak serius:

- Jika terlalu ketat → data valid malah ditolak
- Jika terlalu longgar → duplikasi data yang seharusnya tidak boleh terjadi bisa lolos

Karena itu, sebelum membuatnya, kamu harus benar-benar memahami:

- Status data dalam sistem
- Lifecycle data (dari draft sampai selesai)
- Aturan bisnis yang sebenarnya ingin ditegakkan

---

#### Intuisi Sederhana

Partial unique index pada dasarnya memungkinkan kita mengatakan:

> “Keunikan itu tidak selalu berlaku di semua kondisi, hanya pada keadaan tertentu yang penting bagi bisnis.”

Inilah yang membuatnya sangat berguna untuk sistem dengan banyak status dan workflow kompleks.

## 3. Hubungan Antar Konsep

Pemahaman tentang partial index sangat erat kaitannya dengan konsep **cardinality** dan **selectivity** yang dibahas di video sebelumnya:

- **Cardinality** mengacu pada seberapa banyak nilai unik dalam sebuah kolom.
- **Selectivity** mengacu pada seberapa efektif sebuah indeks menyaring baris.

Ketika data sangat _skewed_ — misalnya 98% baris adalah pengguna non-pro dan hanya 2% adalah pro — membuat full index pada kolom tersebut mungkin kurang efisien karena indeks akan besar dan banyak entri yang tidak berguna. **Partial index hadir sebagai solusi**: hanya indeks subset yang benar-benar kita query, sehingga indeks tetap kecil dan efisien.

Selain itu, partial index membuktikan bahwa desain database yang baik tidak hanya soal struktur tabel, tapi juga soal **memahami pola query dan business logic** secara mendalam.

```
Cardinality/Selectivity tinggi pada subset tertentu
         ↓
Pertimbangkan Partial Index
         ↓
         ├── Untuk performa query → Partial Index biasa (WHERE predicate)
         └── Untuk aturan data   → Partial Unique Index (UNIQUE + WHERE predicate)
```

---

## 4. Kesimpulan

**Partial index** adalah fitur powerful di PostgreSQL (dan SQLite) yang memungkinkan kita membuat indeks hanya pada subset baris yang memenuhi kondisi tertentu. Manfaat utamanya adalah:

- **Efisiensi query**: B-tree lebih kecil → SELECT lebih cepat.
- **Efisiensi write**: INSERT/UPDATE/DELETE tidak perlu memperbarui entri untuk baris yang tidak relevan.
- **Penegakan aturan bisnis**: Partial unique index memungkinkan keunikan yang fleksibel, seperti mendukung soft delete atau status-based constraints.

Kunci penting yang harus diingat:

1. **Sertakan predicate dalam query** agar query planner bisa menggunakan partial index.
2. **Kenali domain bisnis kamu** sebelum membuat partial unique index, karena predicate yang salah bisa menimbulkan masalah.
3. Partial index **tidak sama** dengan composite index — ia adalah indeks satu kolom dengan filter baris.
