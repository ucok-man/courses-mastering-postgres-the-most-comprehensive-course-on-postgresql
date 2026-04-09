# 📝 Network Addresses dan MAC Addresses di PostgreSQL

## 1. Ringkasan Singkat

Materi ini membahas cara menyimpan **network addresses** (alamat IP) dan **MAC addresses** di PostgreSQL menggunakan tipe data khusus. Meskipun tidak terlalu umum, fitur ini sangat berguna ketika dibutuhkan karena lebih efisien dalam hal storage dan query performance dibandingkan menggunakan representasi text biasa. PostgreSQL menyediakan tiga tipe data khusus: `INET` (untuk alamat IP individual), `CIDR` (untuk network ranges), dan `MACADDR`/`MACADDR8` (untuk MAC addresses).

## 2. Konsep Utama

### a. Tipe Data INET

#### Apa itu INET?

Tipe data **INET** di PostgreSQL digunakan untuk menyimpan alamat IP, baik itu **IPv4** maupun **IPv6**. Berbeda dengan tipe data biasa seperti `TEXT`, tipe ini memang dirancang khusus untuk menangani alamat jaringan.

Beberapa hal penting tentang INET:

- Bisa menyimpan **alamat host saja** (misalnya `192.168.1.1`)
- Bisa juga menyimpan alamat dengan **subnet mask** (misalnya `192.168.1.0/24`)
- Lebih **efisien** dalam penyimpanan dan pencarian data dibandingkan menyimpan IP sebagai string biasa

Dengan kata lain, INET bukan hanya soal menyimpan data, tapi juga membuat operasi terkait jaringan jadi lebih optimal.

#### Contoh Pembuatan Tabel

Berikut contoh sederhana membuat tabel dengan kolom bertipe INET:

```sql
CREATE TABLE inet_examples (
    ip_address INET
);
```

Di sini kita membuat tabel `inet_examples` dengan satu kolom `ip_address` yang khusus menyimpan alamat IP.

#### Contoh Data yang Bisa Disimpan

Selanjutnya, kita bisa memasukkan berbagai jenis alamat IP ke dalam tabel tersebut:

```sql
INSERT INTO inet_examples (ip_address) VALUES
    ('192.168.1.1'),              -- Host address saja (IPv4)
    ('192.168.1.0/24'),           -- Dengan subnet mask
    ('2001:0db8::1'),             -- IPv6 address
    ('10.0.0.0/8');               -- IPv4 dengan subnet

SELECT * FROM inet_examples;
```

Penjelasan:

- `192.168.1.1` → alamat host biasa (IPv4)
- `192.168.1.0/24` → alamat jaringan dengan subnet mask
- `2001:0db8::1` → contoh alamat IPv6
- `10.0.0.0/8` → jaringan besar dengan subnet

Saat dilakukan `SELECT`, semua data ini akan ditampilkan dalam format yang konsisten sesuai tipe INET.

#### Efisiensi Storage

Salah satu keunggulan utama INET adalah efisiensi penyimpanan.

Mari kita bandingkan:

```sql
-- Sebagai text: 40 bytes
SELECT pg_column_size(ip_address::TEXT) FROM inet_examples;

-- Sebagai INET: 22 bytes (hampir 50% lebih kecil!)
SELECT pg_column_size(ip_address::INET) FROM inet_examples;
```

Penjelasan:

- Jika disimpan sebagai `TEXT`, alamat IP bisa memakan sekitar **40 byte**
- Jika disimpan sebagai `INET`, hanya sekitar **22 byte**

Artinya, penggunaan INET bisa menghemat hampir **50% ruang penyimpanan**.

Selain hemat storage, tipe INET juga membuat query seperti pencarian subnet atau perbandingan IP menjadi lebih cepat dan efisien karena PostgreSQL memahami struktur data IP tersebut, bukan sekadar teks biasa.

### b. Fungsi-Fungsi untuk INET

#### Gambaran Umum

Selain menyediakan tipe data khusus untuk alamat IP, PostgreSQL juga dilengkapi dengan berbagai fungsi bawaan untuk **memanipulasi dan mengekstrak informasi dari data INET**.

Keuntungan utamanya adalah kita tidak perlu lagi melakukan parsing manual seperti saat menggunakan `TEXT`. PostgreSQL sudah memahami struktur IP address, sehingga kita bisa langsung mengambil bagian-bagian pentingnya dengan fungsi yang tersedia.

#### Fungsi-Fungsi Umum

Berikut beberapa fungsi yang sering digunakan untuk bekerja dengan tipe data INET:

```sql
SELECT
    host(ip_address),           -- Mengambil host saja (tanpa mask)
    masklen(ip_address),        -- Panjang subnet mask
    network(ip_address),        -- Network address
    abbrev(ip_address)          -- Versi abbreviated dari IP
FROM inet_examples;
```

Penjelasan masing-masing fungsi:

- `host(ip_address)`
  Mengambil **alamat IP saja** tanpa subnet mask.
  Contoh: `192.168.1.0/24` → menjadi `192.168.1.0`

- `masklen(ip_address)`
  Mengembalikan **panjang subnet mask** dalam bentuk angka.
  Contoh: `/24` → hasilnya `24`

- `network(ip_address)`
  Menghasilkan **alamat network lengkap dengan subnet mask**
  (ini memastikan formatnya konsisten sebagai network address)

- `abbrev(ip_address)`
  Menampilkan versi **lebih ringkas (abbreviated)** dari alamat IP
  Biasanya menghilangkan bagian yang tidak perlu ditampilkan

#### Contoh Hasil

Jika kita jalankan query di atas, hasilnya akan terlihat seperti ini:

```
host          | masklen | network         | abbrev
--------------+---------+-----------------+--------------
192.168.1.1   | 32      | 192.168.1.1/32  | 192.168.1.1
192.168.1.0   | 24      | 192.168.1.0/24  | 192.168.1/24
2001:db8::1   | 128     | 2001:db8::1/128 | 2001:db8::1
```

Penjelasan dari hasil tersebut:

- IP tanpa subnet default akan dianggap memiliki mask penuh (`/32` untuk IPv4, `/128` untuk IPv6)
- Fungsi `network()` memastikan alamat ditampilkan dalam format network
- Fungsi `abbrev()` bisa membuat tampilan lebih singkat, seperti `192.168.1/24`

#### Keuntungan Menggunakan Fungsi INET

Menggunakan fungsi-fungsi ini memberikan beberapa keuntungan besar:

- Tidak perlu lagi melakukan **parsing string manual** untuk memisahkan bagian IP
- Query menjadi lebih **ringkas, jelas, dan mudah dibaca**
- Performa lebih baik karena PostgreSQL memproses data sebagai struktur jaringan, bukan teks
- Memudahkan operasi lanjutan seperti **pengecekan range IP atau subnet**

Secara keseluruhan, fungsi-fungsi INET ini membuat pengolahan data jaringan di database menjadi jauh lebih praktis dan efisien.

### c. Tipe Data CIDR

#### Apa itu CIDR?

**CIDR (Classless Inter-Domain Routing)** adalah tipe data di PostgreSQL yang digunakan untuk menyimpan **network ranges** atau rentang jaringan.

Berbeda dengan tipe **INET** yang bisa menyimpan alamat IP individu maupun subnet, **CIDR lebih fokus pada representasi jaringan (network)**, bukan host tunggal.

Beberapa poin penting:

- Digunakan khusus untuk **subnet atau network range**
- Tidak cocok untuk menyimpan IP host individual tanpa konteks jaringan
- Sintaks penulisannya mirip dengan INET (misalnya `192.168.1.0/24`)

Dengan kata lain, jika INET itu fleksibel (host + network), maka CIDR itu lebih “strict” untuk urusan network saja.

#### Contoh Pembuatan dan Penggunaan

Berikut contoh sederhana penggunaan tipe data CIDR:

```sql id="c4d2aa"
CREATE TABLE cidr_examples (
    network CIDR
);
```

Di sini kita membuat tabel `cidr_examples` dengan kolom `network` bertipe CIDR.

Selanjutnya, kita isi dengan beberapa contoh network:

```sql id="n8d2k1"
INSERT INTO cidr_examples (network) VALUES
    ('192.168.1.0/24'),         -- IPv4 network
    ('10.0.0.0/8'),             -- Network IPv4 yang lebih besar
    ('2001:db8::/32');          -- IPv6 network

SELECT * FROM cidr_examples;
```

Penjelasan:

- `192.168.1.0/24` → subnet kecil (umumnya untuk jaringan lokal)
- `10.0.0.0/8` → subnet besar (sering dipakai di jaringan privat skala besar)
- `2001:db8::/32` → contoh network IPv6

Saat di-`SELECT`, PostgreSQL akan menampilkan nilai ini dalam format network yang konsisten.

#### Perbedaan INET vs CIDR

Memahami perbedaan INET dan CIDR itu penting supaya tidak salah memilih tipe data.

- **Fokus**
  INET: untuk alamat IP individual (host) maupun network
  CIDR: khusus untuk network ranges

- **Use case**
  INET: menyimpan IP spesifik seperti alamat user atau server
  CIDR: menyimpan subnet seperti `192.168.0.0/16`

- **Storage**
  Keduanya sama-sama disimpan dalam format non-text yang **compact dan efisien**

- **Query**
  INET: sering digunakan untuk mengecek apakah suatu IP termasuk dalam range tertentu
  CIDR: lebih fokus pada operasi antar network (misalnya membandingkan subnet)

Secara praktis:

- Gunakan **INET** jika berurusan dengan IP individual
- Gunakan **CIDR** jika berurusan dengan jaringan atau subnet

#### Kapan Menggunakan CIDR?

CIDR sangat berguna dalam beberapa skenario berikut:

- Saat perlu menyimpan daftar **network ranges** (misalnya whitelist atau blacklist subnet)
- Ketika ingin melakukan pengecekan seperti:
  _“Apakah IP tertentu termasuk dalam network ini?”_
- Untuk kebutuhan **manajemen subnet**, seperti dalam sistem jaringan atau keamanan

Karena PostgreSQL memahami struktur CIDR, operasi seperti pengecekan range bisa dilakukan dengan lebih cepat dan akurat dibandingkan jika menggunakan tipe data teks biasa.

### d. Tipe Data MAC Addresses

#### Apa itu MAC Address di PostgreSQL?

PostgreSQL menyediakan dua tipe data khusus untuk menyimpan **MAC address**, yaitu:

- `MACADDR` → untuk MAC address **6 byte** (format standar)
- `MACADDR8` → untuk MAC address **8 byte** (format EUI-64)

Tipe data ini sangat berguna dalam konteks seperti:

- Network management
- Sistem IoT (Internet of Things)
- Aplikasi yang perlu mengidentifikasi perangkat secara unik

Dibandingkan menyimpan MAC address sebagai `TEXT`, kedua tipe ini jauh lebih **efisien dan hemat storage**, karena disimpan dalam bentuk biner yang compact.

#### Dua Versi MAC Address

Berikut contoh penggunaan kedua tipe tersebut:

```sql id="m1a2c3"
-- MAC Address 6 bytes (standar)
SELECT '08:00:2b:01:02:03'::MACADDR;

-- MAC Address 8 bytes (EUI-64 format)
SELECT '08:00:2b:01:02:03:04:05'::MACADDR8;
```

Penjelasan:

- Format `08:00:2b:01:02:03` adalah MAC address standar (6 byte)
- Format `08:00:2b:01:02:03:04:05` adalah versi lebih panjang (8 byte), biasanya digunakan dalam skema EUI-64

PostgreSQL akan memvalidasi format ini secara otomatis saat casting.

#### Perbandingan Ukuran Storage

Salah satu alasan utama menggunakan tipe khusus ini adalah efisiensi penyimpanan.

```sql id="sz92ka"
-- Sebagai text: 21 bytes
SELECT pg_column_size('08:00:2b:01:02:03'::TEXT);

-- Sebagai MACADDR: 6 bytes
SELECT pg_column_size('08:00:2b:01:02:03'::MACADDR);

-- Sebagai MACADDR8: 8 bytes
SELECT pg_column_size('08:00:2b:01:02:03:04:05'::MACADDR8);
```

Penjelasan:

- Jika disimpan sebagai `TEXT`, membutuhkan sekitar **21 byte**
- Jika menggunakan `MACADDR`, hanya **6 byte**
- Jika menggunakan `MACADDR8`, hanya **8 byte**

Ini menunjukkan penghematan storage yang signifikan, terutama jika data yang disimpan dalam jumlah besar.

#### Konversi Antar Tipe

PostgreSQL juga memungkinkan konversi (casting) antar tipe MAC address:

```sql id="c0nv45"
-- MACADDR (6 bytes) bisa di-cast ke MACADDR8 (8 bytes)
SELECT '08:00:2b:01:02:03'::MACADDR::MACADDR8;
```

Penjelasan:

- MAC address 6 byte bisa dikonversi ke format 8 byte
- Namun sebaliknya perlu perhatian: jika data memang 8 byte, maka **harus menggunakan `MACADDR8`**

Hal penting:

- `MACADDR8` bisa menyimpan baik versi 6 byte maupun 8 byte
- Tapi `MACADDR` hanya bisa menyimpan versi 6 byte

#### Kapan Menggunakan Masing-Masing?

Pemilihan tipe tergantung kebutuhan:

- `MACADDR`
  Gunakan jika hanya bekerja dengan MAC address standar 6 byte
  (misalnya perangkat jaringan umum seperti router atau NIC)

- `MACADDR8`
  Gunakan jika:
  - Berurusan dengan format **EUI-64 (8 byte)**
  - Membutuhkan fleksibilitas untuk menyimpan kedua format

Dengan memilih tipe yang tepat, kamu bisa mendapatkan kombinasi terbaik antara **efisiensi storage, validasi data, dan kemudahan query** dalam pengolahan MAC address.

### e. Prinsip Pemilihan Tipe Data

#### Prinsip Fundamental

Dalam desain database, ada satu prinsip penting yang harus selalu dipegang:

**Pilih tipe data yang paling akurat merepresentasikan realitas**

Artinya, tipe data yang kita gunakan seharusnya mencerminkan bentuk asli dari data tersebut di dunia nyata, bukan sekadar cara termudah untuk menyimpannya.

Berikut gambaran sederhananya:

```
Realitas:        Tipe Data yang Benar:
----------------|-----------------------
IP Address      | INET (bukan TEXT)
Network Range   | CIDR (bukan TEXT)
MAC Address     | MACADDR/MACADDR8 (bukan TEXT)
```

Dengan pendekatan ini, database tidak hanya menyimpan data, tetapi juga “memahami” struktur dan maknanya.

#### Mengapa Ini Penting?

Menggunakan tipe data yang tepat memberikan beberapa keuntungan besar:

1. **Storage lebih efisien**
   Tipe data khusus seperti INET atau MACADDR disimpan dalam format biner yang lebih compact.
   Hasilnya, ukuran data bisa **hingga 50% lebih kecil** dibandingkan TEXT.

2. **Query lebih cepat**
   Jika menggunakan TEXT, database harus melakukan parsing string setiap kali query dijalankan.
   Dengan tipe khusus, PostgreSQL langsung memahami strukturnya sehingga proses jadi lebih cepat.

3. **Operasi built-in tersedia**
   PostgreSQL menyediakan banyak fungsi khusus (seperti `host()`, `network()`, dll) yang hanya bisa digunakan jika tipe datanya sesuai.
   Ini membuat query lebih powerful dan mudah ditulis.

4. **Type safety (validasi otomatis)**
   Database akan memastikan bahwa data yang dimasukkan sesuai format.
   Misalnya:
   - IP address harus valid
   - MAC address harus sesuai format
     Jika tidak valid, PostgreSQL akan menolak input tersebut.

#### Contoh Perbandingan

Mari lihat perbandingan antara pendekatan yang kurang tepat dan yang lebih baik:

```sql id="bad123"
-- ❌ Cara yang kurang efisien
CREATE TABLE devices_bad (
    ip_address TEXT,
    mac_address TEXT
);
```

Pada contoh ini:

- Semua data disimpan sebagai TEXT
- Tidak ada validasi format
- Tidak bisa memanfaatkan fungsi khusus jaringan
- Kurang efisien dalam storage dan query

Sekarang bandingkan dengan pendekatan yang benar:

```sql id="good456"
-- ✅ Cara yang lebih efisien
CREATE TABLE devices_good (
    ip_address INET,
    mac_address MACADDR
);
```

Di sini:

- `ip_address` menggunakan tipe **INET**
- `mac_address` menggunakan tipe **MACADDR**

Hasilnya:

- Data lebih hemat storage
- Query lebih cepat
- Bisa menggunakan fungsi bawaan PostgreSQL
- Data lebih aman karena tervalidasi otomatis

#### Inti yang Perlu Diingat

Saat merancang database, jangan hanya berpikir “yang penting bisa disimpan”.
Sebaliknya, pikirkan:

- Apa bentuk asli datanya?
- Apakah ada tipe data khusus yang lebih cocok?
- Apakah database bisa membantu memvalidasi dan mengolah data tersebut?

Dengan memilih tipe data yang tepat sejak awal, kamu akan mendapatkan sistem yang lebih **efisien, aman, dan mudah dikembangkan ke depannya**.

## 3. Hubungan Antar Konsep

Ketiga tipe data ini saling melengkapi dalam ekosistem network management:

```
Network Infrastructure
         |
         |
    ┌────┴────┐
    |         |
NETWORKS   DEVICES
(CIDR)    (INET + MACADDR)
    |         |
    └────┬────┘
         |
   RANGE QUERIES
```

**Alur penggunaan:**

1. **CIDR** → Mendefinisikan network ranges/subnets yang ada
2. **INET** → Menyimpan IP address dari masing-masing device
3. **MACADDR** → Menyimpan hardware address dari device
4. **Query operations** → Cek apakah device (INET) berada dalam network tertentu (CIDR)

**Contoh use case lengkap:**

```sql
-- Tabel untuk networks
CREATE TABLE networks (
    id SERIAL PRIMARY KEY,
    name TEXT,
    network_range CIDR
);

-- Tabel untuk devices
CREATE TABLE devices (
    id SERIAL PRIMARY KEY,
    hostname TEXT,
    ip_address INET,
    mac_address MACADDR,
    network_id INT REFERENCES networks(id)
);

-- Query: Cari semua devices dalam network tertentu
-- (akan dibahas lebih detail di section querying)
```

**Keuntungan menggunakan tipe data khusus:**

- **Storage**: 40-50% lebih kecil dari TEXT
- **Performance**: Query lebih cepat karena tidak perlu parsing
- **Functionality**: Built-in functions untuk operasi network
- **Validation**: Database otomatis validasi format

## 4. Kesimpulan

- PostgreSQL menyediakan tipe data khusus untuk network addresses: **INET**, **CIDR**, dan **MACADDR/MACADDR8**
- **INET** digunakan untuk menyimpan IP addresses (IPv4/IPv6) dengan atau tanpa subnet mask
- **CIDR** lebih cocok untuk menyimpan network ranges dan operasi subnet
- **MACADDR** (6 bytes) dan **MACADDR8** (8 bytes) untuk MAC addresses
- Menggunakan tipe data khusus memberikan keuntungan:
  - Storage **50% lebih efisien** dibanding TEXT
  - Query **lebih cepat** karena tidak perlu parsing
  - Tersedia **built-in functions** untuk operasi network
  - **Type safety** dan validasi otomatis
- Prinsip penting: **Selalu pilih tipe data yang paling akurat merepresentasikan realitas data**
- Meskipun tidak semua aplikasi membutuhkan tipe data ini, sangat berguna untuk use case network management, IoT, atau monitoring

**Template yang direkomendasikan:**

```sql
CREATE TABLE network_devices (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    device_name TEXT,
    ip_address INET,              -- Untuk IP addresses
    mac_address MACADDR,          -- Untuk MAC addresses
    subnet CIDR                   -- Untuk network ranges
);
```
