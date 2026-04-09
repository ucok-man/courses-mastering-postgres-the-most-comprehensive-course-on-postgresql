# 📝 NaN dan Infinity dalam PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas beberapa keunikan dari tipe data numerik di PostgreSQL, khususnya tentang nilai-nilai spesial seperti NaN (Not a Number) dan Infinity. Materi ini menjelaskan tipe data mana yang mendukung nilai-nilai tersebut, bagaimana perilaku mereka dalam operasi matematika, dan kapan nilai-nilai ini mungkin berguna dalam praktik.

## 2. Konsep Utama

### a. NaN (Not a Number)

NaN adalah singkatan dari _Not a Number_, yaitu sebuah nilai khusus yang digunakan untuk merepresentasikan hasil perhitungan numerik yang **tidak dapat dinyatakan sebagai angka yang valid**. Nilai ini bukan nol, bukan kosong, dan juga bukan error — melainkan sebuah _state_ numerik yang secara eksplisit menyatakan “ini bukan angka”.

Dalam PostgreSQL, NaN sering muncul ketika sistem perlu menyimpan hasil perhitungan yang secara matematis tidak terdefinisi atau tidak bermakna sebagai angka biasa.

#### Karakteristik NaN

Ada beberapa sifat penting dari NaN yang perlu dipahami agar tidak keliru saat menggunakannya dalam query atau logika aplikasi.

#### Tipe data yang mendukung NaN

Tidak semua tipe data numerik mendukung NaN. Di PostgreSQL:

- **NUMERIC** dan **FLOAT** mendukung nilai NaN
- **INTEGER** dan turunannya **tidak mendukung NaN**

Artinya, jika Anda bekerja dengan kolom bertipe `INTEGER`, Anda tidak akan pernah menemukan NaN di sana. Jika membutuhkan representasi “bukan angka”, maka tipe `NUMERIC` atau `FLOAT` adalah pilihan yang tepat.

#### Perilaku kesetaraan (equality)

Secara umum, dalam standar IEEE dan banyak bahasa pemrograman, NaN **tidak pernah sama dengan NaN**. Namun PostgreSQL memiliki perilaku khusus:

- **Semua NaN dianggap sama dengan NaN lainnya**

Ini berarti perbandingan `NaN = NaN` akan menghasilkan `TRUE` di PostgreSQL, berbeda dengan banyak sistem lain. Perilaku ini memudahkan operasi seperti `GROUP BY`, `DISTINCT`, dan perbandingan langsung antar nilai NaN.

#### Perilaku dalam sorting

Saat digunakan dalam operasi pengurutan (`ORDER BY`), PostgreSQL memperlakukan NaN sebagai:

- **Nilai yang sangat besar**
- **Lebih besar dari angka numerik mana pun**

Konsekuensinya, jika Anda mengurutkan data secara ascending, nilai NaN akan muncul di bagian akhir hasil query.

#### Contoh penggunaan NaN

Berikut beberapa contoh konkret untuk memahami bagaimana NaN bekerja di PostgreSQL.

```sql
-- Membuat nilai NaN
SELECT 'NaN'::NUMERIC;
```

Query ini secara eksplisit melakukan casting string `'NaN'` ke tipe `NUMERIC`. Hasilnya adalah sebuah nilai NaN, bukan error.

```sql
-- NaN lebih besar dari angka manapun dalam sorting
SELECT 'NaN'::NUMERIC > 999999999::NUMERIC;
```

Hasil dari query ini adalah `TRUE`. Ini menunjukkan bahwa PostgreSQL menganggap NaN lebih besar daripada angka numerik mana pun, bahkan angka yang sangat besar sekalipun.

```sql
-- Operasi dengan NaN
SELECT 'NaN'::NUMERIC + 1;
```

Hasilnya tetap NaN. Sebagian besar operasi aritmatika yang melibatkan NaN akan menghasilkan NaN, karena menambahkan angka ke “bukan angka” tetap tidak menghasilkan angka yang valid.

```sql
SELECT POWER('NaN'::NUMERIC, 0);
```

Query ini menghasilkan nilai `1`. Hal ini terjadi karena secara matematis, **setiap nilai berpangkat 0 didefinisikan sebagai 1**, termasuk NaN dalam konteks PostgreSQL. Contoh ini penting karena menunjukkan bahwa ada kasus tertentu di mana operasi dengan NaN tidak menghasilkan NaN, tergantung pada aturan matematis yang diterapkan.

Dengan memahami karakteristik ini, Anda bisa menggunakan NaN secara lebih sadar dan aman, terutama saat menangani data numerik yang berasal dari hasil perhitungan kompleks atau sumber data yang tidak selalu valid.

### b. Infinity (Nilai Tak Terbatas)

Infinity digunakan untuk merepresentasikan nilai **tak terbatas (unbounded)**, yaitu nilai yang secara konseptual lebih besar atau lebih kecil dari angka mana pun yang bisa direpresentasikan secara normal. Di PostgreSQL, infinity tersedia dalam dua bentuk: **positif (infinity)** dan **negatif (-infinity)**, dan keduanya diperlakukan sebagai nilai numerik khusus, bukan sebagai error.

Nilai ini sangat berguna ketika Anda perlu menandai batas atas atau batas bawah yang “tidak ada akhirnya”, misalnya dalam rentang waktu, perhitungan matematis ekstrem, atau representasi limit.

#### Tipe data yang mendukung Infinity

Tidak semua tipe data numerik dapat menyimpan infinity. PostgreSQL membedakan dukungan ini secara cukup ketat.

- **NUMERIC tanpa precision dan scale** (sering disebut _open-ended numeric_) mendukung infinity
  Artinya kolom `NUMERIC` yang tidak didefinisikan batas digitnya dapat menyimpan nilai tak terbatas.
- **FLOAT** juga mendukung infinity, mengikuti standar floating-point.

Sebaliknya, ada tipe data yang secara definisi tidak bisa menyimpan infinity.

- **INTEGER** tidak mendukung infinity karena integer selalu memiliki batas minimum dan maksimum yang jelas.
- **NUMERIC dengan precision dan scale tertentu** (_closed numeric_) juga tidak mendukung infinity, karena ruang nilainya dibatasi sejak awal.

Memahami perbedaan ini penting agar tidak terjadi error saat mendesain skema tabel atau melakukan casting nilai.

#### Varian infinity

Infinity hadir dalam dua varian yang saling berlawanan:

- `infinity` untuk nilai tak terbatas positif
- `-infinity` untuk nilai tak terbatas negatif

Keduanya adalah nilai yang valid dan bisa digunakan dalam operasi perbandingan maupun aritmatika, selama tipe datanya mendukung.

#### Perilaku kesetaraan

Dalam PostgreSQL:

- Semua nilai `infinity` dianggap **sama dengan infinity lainnya**
- Semua nilai `-infinity` juga dianggap **sama dengan -infinity lainnya**

Namun tentu saja, `infinity` dan `-infinity` **tidak sama** satu sama lain. Perilaku ini membuat operasi seperti perbandingan, `GROUP BY`, atau pengecekan nilai menjadi lebih konsisten dan mudah diprediksi.

#### Contoh penggunaan Infinity

Berikut beberapa contoh untuk melihat bagaimana infinity bekerja secara praktis.

```sql
-- Membuat nilai infinity
SELECT 'infinity'::NUMERIC;
SELECT '-infinity'::NUMERIC;
```

Kedua query ini melakukan casting string ke tipe `NUMERIC`. Hasilnya adalah nilai infinity positif dan negatif tanpa memicu error, selama tipe `NUMERIC` yang digunakan bersifat open-ended.

```sql
-- Operasi matematika dengan infinity
SELECT 'infinity'::NUMERIC + 'infinity'::NUMERIC;
```

Hasilnya adalah `infinity`. Menjumlahkan dua nilai tak terbatas positif tetap menghasilkan nilai tak terbatas.

```sql
SELECT 'infinity'::NUMERIC - 'infinity'::NUMERIC;
```

Hasil dari operasi ini adalah `NaN`. Secara matematis, pengurangan antara dua nilai tak terbatas tidak memiliki hasil yang terdefinisi, sehingga PostgreSQL merepresentasikannya sebagai _Not a Number_.

```sql
SELECT 'infinity'::NUMERIC + 1;
SELECT 'infinity'::NUMERIC - 1;
```

Kedua operasi ini tetap menghasilkan `infinity`. Menambahkan atau mengurangi angka berhingga dari nilai tak terbatas tidak mengubah sifat ketakterbatasannya.

Dengan memahami bagaimana infinity bekerja, Anda bisa menggunakannya secara tepat untuk merepresentasikan batas ekstrem tanpa perlu menggunakan nilai “dummy” yang berpotensi menyesatkan, sekaligus menjaga logika perhitungan tetap konsisten dan aman.

### c. Perilaku dalam Operasi Matematika

Ketika NaN dan Infinity terlibat dalam operasi matematika, PostgreSQL tidak memperlakukannya seperti angka biasa. Keduanya mengikuti **aturan khusus** yang dirancang agar hasil perhitungan tetap konsisten secara matematis, meskipun tidak selalu intuitif pada pandangan pertama. Memahami aturan ini sangat penting untuk menghindari hasil query yang mengejutkan atau sulit dilacak.

#### Operasi yang melibatkan Infinity

Infinity merepresentasikan nilai yang tidak terbatas, sehingga hasil operasinya mengikuti logika “ketakterbatasan” tersebut.

- `infinity + infinity = infinity`
  Menjumlahkan dua nilai tak terbatas positif tetap menghasilkan nilai tak terbatas. Tidak ada angka berhingga yang bisa membatasi hasilnya.

- `infinity - infinity = NaN`
  Operasi ini menghasilkan NaN karena secara matematis tidak terdefinisi. Kita tidak bisa menentukan hasil pasti dari pengurangan dua nilai yang sama-sama tak terbatas.

- `infinity + 1 = infinity`
  Menambahkan angka berhingga ke nilai tak terbatas tidak mengubah sifat ketakterbatasannya. Hasilnya tetap infinity.

- `infinity - 1 = infinity`
  Hal yang sama berlaku saat mengurangi angka berhingga dari infinity. Nilai tersebut masih tetap tak terbatas.

Aturan-aturan ini membantu PostgreSQL menjaga konsistensi logika matematika saat berhadapan dengan nilai ekstrem.

#### Operasi yang melibatkan NaN

NaN melambangkan nilai yang bukan angka valid, sehingga hampir semua operasi yang melibatkannya akan “tercemar” oleh NaN itu sendiri.

- `NaN + angka_apapun = NaN`
  Jika sebuah operasi melibatkan NaN, hasilnya hampir selalu NaN. Ini menandakan bahwa perhitungan tersebut tidak memiliki hasil numerik yang bermakna.

- `NaN ^ 0 = 1`
  Ini adalah pengecualian penting. Secara matematis, aturan umum menyatakan bahwa **setiap nilai berpangkat nol adalah 1**. PostgreSQL menerapkan aturan ini bahkan ketika basisnya adalah NaN, sehingga hasilnya tetap 1.

Pengecualian ini sering menjadi sumber kebingungan jika tidak dipahami dengan baik.

#### Contoh perilaku dalam query

Contoh berikut menunjukkan langsung bagaimana PostgreSQL menerapkan aturan-aturan tersebut.

```sql
-- Infinity
SELECT 'infinity'::NUMERIC + 'infinity'::NUMERIC;
```

Hasilnya adalah `infinity`, karena penjumlahan dua nilai tak terbatas tetap tak terbatas.

```sql
SELECT 'infinity'::NUMERIC - 'infinity'::NUMERIC;
```

Query ini menghasilkan `NaN`, menandakan bahwa operasi tersebut tidak memiliki hasil yang terdefinisi secara matematis.

```sql
-- NaN
SELECT 'NaN'::NUMERIC + 1;
```

Hasilnya adalah `NaN`. Kehadiran NaN dalam operasi membuat seluruh hasil perhitungan menjadi NaN.

```sql
SELECT POWER('NaN'::NUMERIC, 0);
```

Hasilnya adalah `1`. Ini menunjukkan pengecualian penting di mana aturan matematika `x⁰ = 1` tetap berlaku, bahkan ketika `x` adalah NaN.

Dengan memahami perilaku ini, Anda dapat menulis query yang lebih aman dan dapat diprediksi, terutama saat menangani data numerik yang berpotensi mengandung nilai ekstrem atau hasil perhitungan yang tidak terdefinisi.

## 3. Hubungan Antar Konsep

- **NaN dan Infinity adalah nilai spesial** yang hanya tersedia pada tipe data numerik yang tidak terbatas (unbounded)
- **INTEGER tidak mendukung keduanya** karena integer secara definisi memiliki batas minimum dan maksimum
- **NUMERIC dengan precision dan scale** juga tidak mendukung infinity karena range-nya sudah ditentukan
- **Dalam operasi matematika**, NaN cenderung "menginfeksi" hasil perhitungan (operasi apapun dengan NaN menghasilkan NaN), kecuali kasus khusus seperti pangkat nol
- **Infinity lebih stabil** dalam operasi matematika dan mengikuti logika matematika (infinity ± bilangan finite tetap infinity)

## 4. Kesimpulan

NaN dan Infinity adalah nilai-nilai spesial dalam PostgreSQL yang hanya tersedia pada tipe data `NUMERIC` (tanpa precision/scale) dan `FLOAT`. Nilai-nilai ini tidak tersedia pada `INTEGER` atau `NUMERIC` dengan range yang ditentukan.

Infinity lebih berguna dalam praktik untuk merepresentasikan nilai yang tidak terbatas, sementara NaN lebih sering muncul sebagai hasil operasi yang tidak valid. Keduanya memiliki perilaku khusus dalam operasi matematika dan sorting yang penting untuk dipahami saat bekerja dengan data numerik di PostgreSQL.

**Key takeaway:** Pahami batasan tipe data yang digunakan - hanya unbounded numeric types yang mendukung NaN dan Infinity.
