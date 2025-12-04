# Questions

- <details>
  <summary>Apa yang dimaksud dengan "heap" di posgresql?</summary> <br>

  PostgreSQL memiliki struktur data heap sendiri, tapi penting untuk memahami bahwa "heap" di sini berbeda dengan struktur data heap dalam algoritma (seperti binary heap untuk priority queue).

  ## Heap Storage PostgreSQL

  Dalam PostgreSQL, **"heap" merujuk pada format penyimpanan default untuk tabel**. Ketika Anda membuat tabel tanpa menentukan format khusus, PostgreSQL akan menyimpannya sebagai "heap table". Berikut penjelasannya:

  ### Karakteristik Heap PostgreSQL:

  - **Tidak ada urutan tertentu**: Baris-baris data disimpan tanpa urutan khusus (meskipun urutan fisik awalnya mungkin mengikuti urutan insert)
  - **Berbasis page**: Data diorganisir dalam page dengan ukuran tetap (biasanya 8KB)
  - **Mendukung MVCC**: Struktur heap mendukung Multi-Version Concurrency Control PostgreSQL, menyimpan beberapa versi dari baris yang sama
  - **Append-mostly**: Baris baru umumnya ditambahkan ke ruang kosong di page yang ada atau page baru

  Disebut "heap" karena mirip seperti tumpukan barang yang tidak memiliki struktur atau urutan tertentu.

  ## Implementasi dalam Kode

  Implementasi heap PostgreSQL dapat ditemukan di source code, terutama di direktori `src/backend/access/heap/`, dengan file-file seperti:

  - `heapam.c` - metode akses heap
  - `heaptuple.c` - operasi tuple heap
  - `hio.c` - operasi I/O heap

  ## Metode Penyimpanan Alternatif

  PostgreSQL juga mendukung format penyimpanan lain melalui table access methods (diperkenalkan di PostgreSQL 12), meskipun heap tetap menjadi default dan paling umum digunakan.

  Jadi ya, PostgreSQL memiliki struktur heap sendiri, tetapi ini adalah format penyimpanan bukan struktur data algoritmik.

  </details>

- <details>
  <summary>Apa yang dimaksud dengan "pages" di posgresql?</summary>
  <br>

  **Pages** (atau sering disebut **blocks**) adalah unit dasar penyimpanan data di PostgreSQL. Mari saya jelaskan secara detail:

  ## Definisi Pages

  Pages adalah **blok memori berukuran tetap** yang digunakan PostgreSQL untuk menyimpan data di disk dan membacanya ke memori. Ini seperti "halaman" dalam sebuah buku.

  ### Karakteristik Pages:

  - **Ukuran default: 8KB** (8192 bytes)
  - Ukuran ini tetap dan ditentukan saat kompilasi PostgreSQL
  - Semua operasi I/O (baca/tulis) dilakukan dalam satuan page
  - Setiap page memiliki struktur internal yang terorganisir

  ## Struktur Internal Page

  Setiap page dibagi menjadi beberapa bagian:

  1. **Page Header** - informasi metadata (24 bytes)
  2. **Item Pointers** - array pointer ke tuple/baris
  3. **Free Space** - ruang kosong di tengah
  4. **Tuples/Rows** - data baris aktual (disimpan dari bawah ke atas)
  5. **Special Space** - data khusus untuk indeks (opsional)

  ```
  ┌─────────────────────┐
  │   Page Header       │ ← Metadata
  ├─────────────────────┤
  │   Item Pointers     │ ← Pointer ke tuple
  ├─────────────────────┤
  │                     │
  │   Free Space        │ ← Ruang kosong
  │                     │
  ├─────────────────────┤
  │   Tuple/Row Data    │ ← Data aktual
  └─────────────────────┘
  ```

  ## Kenapa Menggunakan Pages?

  1. **Efisiensi I/O**: Disk membaca/menulis data dalam blok, bukan byte per byte
  2. **Buffer Management**: PostgreSQL bisa cache pages di memori (shared buffers)
  3. **Atomicity**: Operasi pada satu page bisa dijamin atomic
  4. **Konsistensi**: Ukuran tetap memudahkan manajemen storage

  ## Contoh Praktis

  Misalnya Anda punya tabel dengan baris berukuran 200 bytes:

  - Satu page (8KB) bisa menampung sekitar **40 baris**
  - Jika tabel punya 1000 baris, butuh sekitar **25 pages**
  - Ketika query mencari data, PostgreSQL membaca page demi page ke memori

  ## Melihat Informasi Pages

  Anda bisa melihat berapa banyak page yang digunakan tabel:

  ```sql
  SELECT relpages, reltuples
  FROM pg_class
  WHERE relname = 'nama_tabel';
  ```

  - `relpages` = jumlah pages
  - `reltuples` = estimasi jumlah baris

  Jadi, **pages adalah container penyimpanan dasar** tempat PostgreSQL menyimpan semua data tabel dan indeks Anda.

    </details>
