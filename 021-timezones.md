# 📝 Time Zones di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang penanganan **time zones** (zona waktu) di PostgreSQL, yang merupakan salah satu topik yang paling membingungkan dalam pemrograman database. Pembahasan mencakup best practices dalam menyimpan dan menampilkan waktu, perbedaan antara named time zones dan offset, serta jebakan umum yang harus dihindari saat bekerja dengan zona waktu.

## 2. Konsep Utama

### a. Kompleksitas Time Zones

Time zones sering dianggap sekadar perbedaan jam antar wilayah, padahal kenyataannya jauh lebih kompleks. Kompleksitas ini muncul karena aturan time zone **ditentukan oleh kebijakan manusia dan keputusan politik**, bukan oleh perhitungan matematis yang konsisten. Akibatnya, satu wilayah bisa memiliki aturan waktu yang berbeda dari wilayah lain, bahkan bisa berubah dari waktu ke waktu.

Di bawah ini adalah beberapa faktor utama yang membuat pengelolaan time zone menjadi rumit, terutama dalam konteks penyimpanan dan konversi waktu di sistem komputer.

#### Daylight Saving Time (DST)

Daylight Saving Time adalah praktik memajukan jam (biasanya satu jam) pada periode tertentu, lalu mengembalikannya ke waktu normal di periode lain. Masalahnya, **tidak semua negara atau wilayah menerapkan DST**.

Bahkan di antara wilayah yang menerapkan DST:

- Tanggal mulai dan berakhirnya DST bisa berbeda
- Aturan DST dapat berubah sewaktu-waktu karena kebijakan pemerintah

Sebagai contoh, Amerika Serikat dan negara-negara di Eropa sama-sama menerapkan DST, tetapi tanggal peralihannya tidak selalu sama. Jika sebuah aplikasi melakukan konversi waktu lintas negara tanpa memperhatikan aturan ini, hasilnya bisa meleset satu jam.

#### Perubahan Offset terhadap UTC

Setiap time zone memiliki offset terhadap UTC (Coordinated Universal Time), misalnya `UTC+7` atau `UTC-5`. Namun offset ini **tidak selalu tetap**.

Ketika DST aktif:

- Offset sebuah wilayah bisa berubah
- Contohnya, wilayah yang biasanya `UTC-5` bisa menjadi `UTC-4` selama DST

Artinya, jika sebuah timestamp hanya menyimpan offset tanpa konteks apakah DST sedang berlaku atau tidak, kita bisa salah menafsirkan waktu sebenarnya.

Sebagai ilustrasi:

- `2024-03-10 01:30` di suatu wilayah bisa memiliki arti yang berbeda sebelum dan sesudah DST aktif
- Bahkan ada jam tertentu yang “tidak pernah ada” atau “muncul dua kali” saat transisi DST

#### Ambiguitas Saat Konversi Waktu

Ambiguitas muncul ketika sistem mencoba mengonversi waktu tanpa informasi yang cukup. Untuk melakukan konversi dengan benar, kita perlu tahu:

- Time zone asal
- Apakah waktu tersebut berada dalam periode DST atau non-DST
- Aturan time zone yang berlaku pada tanggal tersebut

Tanpa informasi ini, satu nilai waktu bisa ditafsirkan dengan lebih dari satu cara. Inilah alasan mengapa menyimpan waktu dalam format lokal (misalnya hanya `2024-11-03 01:30`) sangat berisiko, terutama di wilayah yang menerapkan DST.

Karena berbagai kompleksitas ini, praktik yang umum dan aman adalah:

- Menyimpan waktu dalam UTC
- Baru mengonversinya ke time zone lokal saat ditampilkan ke pengguna

Dengan pendekatan ini, sistem terhindar dari sebagian besar masalah yang disebabkan oleh DST, perubahan offset, dan ambiguitas waktu.

### b. Best Practice #1: Simpan Semuanya dalam UTC

Praktik terbaik yang paling sering direkomendasikan dalam pengelolaan waktu adalah **menyimpan semua timestamp dalam UTC (Coordinated Universal Time)**. Prinsipnya sederhana: selama data masih diproses, disimpan, atau dikirim antar sistem, gunakan UTC. Konversi ke zona waktu lokal hanya dilakukan **di tahap paling akhir**, yaitu saat data akan ditampilkan ke pengguna.

Pendekatan ini membantu menghindari berbagai masalah klasik seperti perbedaan offset, perubahan DST, dan ambiguitas waktu. Dengan UTC sebagai standar internal, sistem menjadi lebih konsisten dan mudah diprediksi, terutama ketika aplikasi digunakan oleh user dari berbagai zona waktu.

#### Mengapa UTC Menjadi Pilihan Terbaik

UTC tidak terpengaruh oleh Daylight Saving Time dan tidak berubah-ubah sepanjang tahun. Artinya:

- Satu timestamp selalu merepresentasikan satu momen yang sama di seluruh dunia
- Tidak ada jam “maju” atau “mundur” seperti pada zona waktu lokal
- Konversi ke waktu lokal bisa dilakukan kapan saja dengan aturan yang jelas

Karena itu, menyimpan waktu dalam UTC selama mungkin adalah fondasi penting untuk sistem yang andal.

#### Cara Mengatur Time Zone di PostgreSQL

PostgreSQL menyediakan beberapa cara untuk mengatur time zone, tergantung pada kebutuhan dan cakupan pengaruhnya.

Untuk melihat time zone yang sedang aktif, kita bisa menjalankan:

```sql
SHOW timezone;
```

Perintah ini akan menampilkan zona waktu yang digunakan oleh session saat ini. Hasilnya bisa berupa `UTC`, `Asia/Jakarta`, `America/Chicago`, atau zona waktu lainnya.

Jika ingin mengubah time zone **hanya untuk session yang sedang aktif**, gunakan:

```sql
SET timezone = 'America/Chicago';
```

Perubahan ini langsung berlaku, tetapi hanya untuk koneksi tersebut. Cocok digunakan untuk kebutuhan sementara, misalnya saat debugging atau menjalankan query tertentu.

Untuk pengaturan yang lebih permanen di level database, gunakan:

```sql
ALTER DATABASE demo SET timezone TO 'UTC';
```

Dengan konfigurasi ini, setiap koneksi baru ke database `demo` akan otomatis menggunakan UTC sebagai default time zone, kecuali diubah lagi di level session.

#### Hirarki Pengaturan Time Zone

PostgreSQL memiliki hirarki pengaturan time zone, dari yang paling umum hingga yang paling spesifik:

1. **Cluster level**
   Diatur di file konfigurasi PostgreSQL (`postgresql.conf`). Pengaturan ini berlaku untuk seluruh database dalam satu cluster.

2. **Database level**
   Diatur menggunakan perintah `ALTER DATABASE`. Berlaku untuk satu database tertentu.

3. **Session level**
   Diatur menggunakan perintah `SET timezone`. Hanya berlaku untuk koneksi yang sedang aktif.

PostgreSQL akan selalu menggunakan pengaturan yang **paling spesifik**. Jika time zone diset di level session, maka nilai tersebut akan menimpa setting di level database maupun cluster.

#### Catatan Penting tentang Session

Perlu diingat bahwa pengaturan time zone di level session bersifat sementara. Saat koneksi terputus atau aplikasi membuat koneksi baru, setting ini akan direset dan kembali menggunakan konfigurasi di level database atau cluster.

Karena itu, untuk aplikasi produksi, sangat disarankan:

- Mengatur time zone database ke UTC
- Menghindari ketergantungan pada setting session kecuali benar-benar diperlukan

Dengan mengikuti praktik ini, sistem akan lebih stabil dan aman dalam menangani data waktu lintas zona waktu.

### c. Perilaku `TIMESTAMP WITH TIME ZONE` (TIMESTAMPTZ)

Di PostgreSQL, tipe data `TIMESTAMP WITH TIME ZONE` (sering disingkat `TIMESTAMPTZ`) memiliki perilaku khusus yang sangat penting untuk dipahami. Tipe data ini **bukan sekadar menyimpan tanggal dan jam**, tetapi juga melibatkan mekanisme konversi time zone secara otomatis.

Inti dari perilaku `TIMESTAMPTZ` adalah:

- PostgreSQL **selalu menyimpan nilainya dalam UTC**
- Time zone hanya berpengaruh **saat data ditampilkan atau saat data dimasukkan**

Dengan kata lain, time zone tidak benar-benar “menempel” pada data yang disimpan, melainkan digunakan sebagai konteks untuk konversi.

#### Apa yang Terjadi Saat Data Masuk

Ketika kita memasukkan nilai ke kolom bertipe `TIMESTAMPTZ`, PostgreSQL akan:

1. Menentukan time zone asal nilai tersebut
2. Mengonversinya ke UTC
3. Menyimpan hasil konversi tersebut di database

Jika nilai yang dimasukkan **tidak menyertakan informasi time zone**, PostgreSQL akan menganggap nilai tersebut berada di **time zone session yang sedang aktif**.

Contoh saat session berada di UTC:

```sql
SET timezone = 'UTC';
SELECT '2025-01-31 11:30:00'::TIMESTAMPTZ;
```

Karena session menggunakan UTC, PostgreSQL menganggap `11:30` adalah waktu UTC dan menyimpannya apa adanya. Hasil yang ditampilkan:

```
2025-01-31 11:30:00+00
```

Tanda `+00` menunjukkan bahwa waktu tersebut berada di UTC.

#### Apa yang Terjadi Saat Data Keluar

Saat data `TIMESTAMPTZ` dibaca atau ditampilkan, PostgreSQL akan:

1. Mengambil nilai yang disimpan dalam UTC
2. Mengonversinya ke time zone session saat ini
3. Menampilkan hasil konversi tersebut ke client

Sekarang bandingkan dengan session yang menggunakan time zone berbeda:

```sql
SET timezone = 'America/Chicago';
SELECT '2025-01-31 11:30:00'::TIMESTAMPTZ;
```

Di sini, PostgreSQL menganggap `11:30` adalah waktu lokal Chicago. Pada tanggal tersebut, Chicago berada di offset `-06`. PostgreSQL kemudian mengonversinya ke UTC untuk penyimpanan, lalu langsung menampilkannya kembali sesuai time zone session. Hasilnya:

```
2025-01-31 05:30:00-06
```

Perhatikan bahwa jamnya berubah menjadi `05:30`, tetapi momen waktunya **tetap sama secara global**. Perubahan ini murni karena perbedaan time zone saat ditampilkan.

#### Kesimpulan Penting tentang TIMESTAMPTZ

Beberapa poin penting yang perlu diingat:

- `TIMESTAMPTZ` **selalu disimpan dalam UTC**
- Time zone session menentukan bagaimana nilai literal ditafsirkan saat input
- Time zone session juga menentukan bagaimana nilai ditampilkan saat output
- Jika literal tidak memiliki penanda time zone, PostgreSQL menggunakan time zone session sebagai asumsi

Memahami perilaku ini sangat penting agar kita tidak keliru menafsirkan data waktu, terutama ketika aplikasi berjalan di berbagai zona waktu atau melibatkan banyak user dari lokasi berbeda.

### d. Best Practice #2: Gunakan Named Time Zones

Dalam pengelolaan waktu, ada satu aturan yang **sangat penting dan tidak boleh diabaikan**: selalu gunakan **named time zones** seperti `'America/Chicago'`, dan hindari penggunaan offset statis seperti `'-06:00'`.

Sekilas, offset terlihat lebih sederhana. Namun dalam praktik, pendekatan ini justru berisiko dan mudah menimbulkan bug, terutama pada sistem yang berjalan sepanjang tahun dan melibatkan banyak wilayah.

#### Mengapa Harus Menggunakan Named Time Zones

Ada dua alasan utama mengapa named time zones jauh lebih aman dan andal.

Pertama, **named time zones secara otomatis menangani Daylight Saving Time (DST)**. Ketika sebuah wilayah masuk atau keluar dari periode DST, PostgreSQL sudah mengetahui aturan perubahannya. Kita tidak perlu mengubah apa pun di kode atau query.

Sebaliknya, offset seperti `-06:00` bersifat statis. Offset ini tidak tahu apakah suatu tanggal berada di periode DST atau tidak. Akibatnya, konversi waktu bisa meleset satu jam tanpa disadari.

Kedua, penggunaan offset sangat rentan terhadap **kesalahan tanda positif dan negatif**. Kesalahan kecil seperti menulis `+06:00` alih-alih `-06:00` bisa menyebabkan waktu bergeser hingga 12 jam ke arah yang salah. Kesalahan semacam ini sering sulit dideteksi karena tidak selalu langsung terlihat.

Dengan named time zones, risiko ini praktis hilang karena konversi mengikuti aturan resmi zona waktu tersebut.

#### Contoh Konversi ke Time Zone Lokal

Berikut contoh bagaimana melakukan konversi dari UTC ke time zone lokal menggunakan named time zone di PostgreSQL.

Pertama, pastikan session menggunakan UTC agar input timestamp ditafsirkan dengan jelas:

```sql
SET timezone = 'UTC';
```

Kemudian lakukan query konversi:

```sql
SELECT
  '2025-01-31 11:30:00'::TIMESTAMPTZ AS utc_time,
  '2025-01-31 11:30:00'::TIMESTAMPTZ
    AT TIME ZONE 'America/Chicago' AS chicago_time;
```

Pada query ini:

- Kolom `utc_time` menampilkan timestamp dalam UTC
- Ekspresi `AT TIME ZONE 'America/Chicago'` mengonversi waktu tersebut ke waktu lokal Chicago

#### Penjelasan Output

Hasil query di atas adalah:

```
utc_time: 2025-01-31 11:30:00+00
chicago_time: 2025-01-31 05:30:00
```

Nilai `utc_time` jelas menunjukkan bahwa timestamp berada di UTC (`+00`).

Sementara itu, `chicago_time` menunjukkan jam `05:30`, yang merupakan hasil konversi dari UTC ke waktu lokal Chicago pada tanggal tersebut. Perhatikan bahwa hasil ini **tidak memiliki time zone identifier**.

Hal ini terjadi karena operasi `AT TIME ZONE` menghasilkan tipe data **`TIMESTAMP WITHOUT TIME ZONE`**.

#### Catatan Penting tentang `AT TIME ZONE`

Hasil dari `AT TIME ZONE` sebaiknya:

- Hanya digunakan di tahap akhir pipeline
- Ditujukan untuk keperluan display ke user
- Tidak disimpan kembali ke database sebagai sumber kebenaran waktu

Praktik yang aman adalah tetap menyimpan waktu dalam `TIMESTAMPTZ` (UTC) di database, lalu melakukan konversi ke named time zone lokal **hanya saat data akan ditampilkan**. Dengan pendekatan ini, sistem tetap konsisten, bebas dari masalah DST, dan minim risiko kesalahan konversi.

### e. Jebakan: Penggunaan Hour Offset

PostgreSQL memang fleksibel dalam cara menentukan time zone, tetapi justru di sinilah banyak jebakan tersembunyi. Ada **tiga cara** untuk menspesifikasikan time zone, dan tidak semuanya aman untuk digunakan dalam praktik sehari-hari.

#### Tiga Cara Menentukan Time Zone di PostgreSQL

PostgreSQL mendukung tiga pendekatan berikut:

1. **Full time zone name**
   Contoh: `'America/Chicago'`
   Ini adalah pendekatan **yang paling direkomendasikan**. Named time zone mengikuti database aturan resmi (IANA time zone database), sehingga otomatis menangani DST dan perubahan kebijakan waktu.

2. **Time zone abbreviation**
   Contoh: `'CST'`, `'CDT'`
   Pendekatan ini harus digunakan dengan sangat hati-hati. Singkatan seperti `CST` bisa berarti zona waktu yang berbeda di wilayah lain, dan sering kali **tidak otomatis menangani DST**. Jika digunakan, developer harus benar-benar paham konteksnya.

3. **POSIX-style offset**
   Contoh: `'-06:00'`
   Ini adalah pendekatan **yang paling berbahaya** dan sebaiknya dihindari, kecuali benar-benar memahami implikasinya.

#### Masalah Utama dengan POSIX-style Offset

Masalah terbesar dari POSIX-style offset adalah perbedaan konvensi tanda dengan standar internasional yang umum digunakan.

Menurut dokumentasi PostgreSQL:

- **ISO 8601 (standar internasional)**
  Wilayah di barat Greenwich menggunakan tanda **negatif** (`-`).
  Contoh: Chicago → `-06:00`

- **POSIX**
  Wilayah di barat Greenwich justru menggunakan tanda **positif** (`+`).

Perbedaan ini sangat tidak intuitif. Banyak developer mengira `-06:00` berarti “UTC dikurangi 6 jam”, padahal dalam konteks POSIX di PostgreSQL, maknanya justru bisa terbalik.

#### Contoh Kesalahan yang Sangat Umum

Perhatikan contoh berikut:

```sql
SET timezone = 'UTC';

SELECT
  '2025-01-31 11:30:00'::TIMESTAMPTZ AS base,
  '2025-01-31 11:30:00'::TIMESTAMPTZ
    AT TIME ZONE 'America/Chicago' AS named,
  '2025-01-31 11:30:00'::TIMESTAMPTZ
    AT TIME ZONE '-06:00' AS offset_wrong,
  '2025-01-31 11:30:00'::TIMESTAMPTZ
    AT TIME ZONE INTERVAL '-6 hours' AS interval_ok;
```

Mari kita bahas hasilnya satu per satu.

- Menggunakan `'America/Chicago'`
  Hasilnya adalah `05:30`. Ini benar, karena Chicago berada 6 jam di belakang UTC pada tanggal tersebut. PostgreSQL menghitungnya sebagai `11:30 - 6 jam`.

- Menggunakan `'-06:00'`
  Hasilnya justru `17:30`. Ini **salah 12 jam** dari yang diharapkan. PostgreSQL menafsirkan offset ini menggunakan aturan POSIX, sehingga dianggap sebagai `11:30 + 6 jam`.

- Menggunakan `INTERVAL '-6 hours'`
  Hasilnya kembali `05:30`. Pendekatan ini mengikuti logika ISO 8601, sehingga hasil konversinya sesuai ekspektasi.

#### Kesimpulan Penting

Beberapa pelajaran penting dari contoh ini:

- Jangan gunakan POSIX-style offset seperti `'-06:00'` untuk konversi waktu
- Named time zone adalah pilihan paling aman dan konsisten
- Jika benar-benar perlu offset manual, gunakan `INTERVAL '-x hours'`, bukan string offset

Kesalahan tanda kecil seperti ini bisa menyebabkan bug serius yang sulit dilacak, terutama pada sistem yang memproses data waktu lintas zona dan lintas negara.

### f. Melihat Daftar Time Zones

PostgreSQL sudah menyediakan daftar lengkap time zone yang bisa digunakan secara langsung, sehingga kita tidak perlu menebak atau menghafal nama zona waktu yang valid. Daftar ini tersedia dalam bentuk **view sistem**, yang bisa di-query seperti tabel biasa.

View ini sangat berguna ketika:

- Ingin memastikan nama time zone yang digunakan benar
- Mencari time zone tertentu berdasarkan nama wilayah
- Memahami bagaimana PostgreSQL merepresentasikan DST dan offset untuk suatu zona

#### Melihat Semua Time Zone yang Tersedia

Untuk melihat seluruh time zone yang dikenali oleh PostgreSQL, gunakan query berikut:

```sql
SELECT * FROM pg_timezone_names;
```

Query ini akan menampilkan semua named time zones yang tersedia, termasuk zona berbasis wilayah (seperti `America/Chicago`) maupun zona lainnya.

Namun karena jumlahnya cukup banyak, biasanya kita akan memfilter hasilnya.

#### Memfilter Time Zone Berdasarkan Nama

Jika kita hanya tertarik pada time zone tertentu, misalnya yang berkaitan dengan Chicago, kita bisa memfilter menggunakan `ILIKE`:

```sql
SELECT * FROM pg_timezone_names
WHERE name ILIKE '%chicago%';
```

Penggunaan `ILIKE` membuat pencarian tidak case-sensitive, sehingga lebih fleksibel saat mencari nama zona waktu.

#### Memahami Struktur Output

Contoh hasil query tersebut adalah:

```
name              | abbrev | utc_offset | is_dst
------------------+--------+------------+--------
America/Chicago   | CDT    | -05:00     | t
```

Setiap kolom memiliki makna penting:

- `name`
  Nama lengkap time zone. Inilah nilai yang sebaiknya digunakan saat mengatur atau mengonversi time zone, misalnya `'America/Chicago'`.

- `abbrev`
  Singkatan time zone yang sedang aktif saat ini, seperti `CST` atau `CDT`. Nilai ini **bisa berubah** tergantung apakah zona tersebut sedang berada dalam periode Daylight Saving Time atau tidak.

- `utc_offset`
  Selisih waktu terhadap UTC pada saat ini. Sama seperti `abbrev`, nilai ini juga **dinamis** dan akan berubah otomatis ketika masuk atau keluar dari DST.

- `is_dst`
  Menunjukkan apakah zona waktu tersebut sedang berada dalam periode DST (`t` untuk true, `f` untuk false).

#### Hal Penting yang Perlu Diingat

Kolom `abbrev` dan `utc_offset` **bukan nilai statis**. PostgreSQL akan menyesuaikannya secara otomatis berdasarkan tanggal dan aturan DST yang berlaku. Karena itu:

- Jangan mengandalkan nilai `abbrev` atau `utc_offset` sebagai identitas utama time zone
- Selalu gunakan kolom `name` sebagai referensi time zone yang aman dan konsisten

Dengan memahami view ini, kita bisa lebih percaya diri dalam memilih dan menggunakan time zone yang tepat di PostgreSQL.

## 3. Hubungan Antar Konsep

Alur kerja yang benar dalam menangani time zones:

1. **Storage**: Simpan semua timestamp sebagai `TIMESTAMPTZ` dalam UTC
2. **Konfigurasi**: Set database/session ke UTC untuk menghindari konversi otomatis yang tidak diinginkan
3. **Processing**: Lakukan semua operasi date/time dalam UTC
4. **Display**: Hanya di tahap akhir, gunakan `AT TIME ZONE` dengan **named time zone** untuk konversi ke zona waktu lokal user
5. **Never**: Jangan gunakan bare hour offset (`'-06:00'`) karena bisa salah arah

**Diagram Alur:**

```
User Input → Convert to UTC → Store in DB (UTC)
                                    ↓
                            Process in UTC
                                    ↓
                    Retrieve as UTC from DB
                                    ↓
              AT TIME ZONE 'America/Chicago'
                                    ↓
                          Display to User
```

## 4. Kesimpulan

Time zones di PostgreSQL memang kompleks, tapi bisa diatasi dengan mengikuti dua prinsip utama:

1. **Simpan dalam UTC, konversi di akhir**: Keep everything in UTC as long as possible, convert to user's local time zone only at display time
2. **Gunakan named time zones**: Always use named time zones like `'America/Chicago'`, never use bare hour offsets like `'-06:00'`

Dengan mengikuti prinsip ini, Anda akan terhindar dari bug yang umum terjadi seperti:

- Salah arah konversi (12 jam off)
- Tidak handle DST dengan benar
- Konversi ganda yang tidak diinginkan

**Remember**: PostgreSQL's time zone handling is powerful, but you need to be explicit and careful. When in doubt, stay in UTC!
