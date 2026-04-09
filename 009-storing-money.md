# 📝 Menyimpan Data Uang (Money) di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas cara yang **benar dan salah** untuk menyimpan data uang (money) di PostgreSQL. Meskipun PostgreSQL memiliki tipe data `MONEY`, video ini **sangat menyarankan untuk TIDAK menggunakannya** karena berbagai masalah serius. Sebagai gantinya, direkomendasikan dua pendekatan: menyimpan sebagai **INTEGER** (dalam unit terkecil seperti cents) atau sebagai **NUMERIC** dengan precision dan scale yang tepat.

## 2. Konsep Utama

### a. Tipe Data MONEY: Kenapa TIDAK Direkomendasikan

PostgreSQL memang menyediakan tipe data khusus bernama `MONEY`, dan sekilas tipe ini terlihat sangat menarik karena langsung merepresentasikan nilai uang lengkap dengan simbol mata uang. Namun, di balik kemudahan tersebut, ada sejumlah masalah serius yang membuat tipe data ini **sangat tidak disarankan untuk digunakan di lingkungan production**.

#### Deklarasi dan Penggunaan Dasar

Secara sintaks, penggunaan tipe data `MONEY` sangat sederhana. Kita bisa mendeklarasikannya langsung pada kolom tabel seperti contoh berikut:

```sql
CREATE TABLE money_example (
    item_name TEXT,
    price MONEY
);
```

Kolom `price` didefinisikan menggunakan tipe `MONEY`, yang artinya PostgreSQL akan memperlakukan nilai tersebut sebagai nilai moneter.

Selanjutnya, kita bisa memasukkan data dengan berbagai format input:

```sql
INSERT INTO money_example (item_name, price) VALUES
    ('Laptop', 999.99),           -- Float literal
    ('Mouse', 25.50),             -- Decimal literal
    ('Keyboard', 75),             -- Integer literal
    ('Headphones', '$129.99');    -- Currency-formatted string
```

Semua baris di atas **valid** dan akan diterima oleh PostgreSQL tanpa error. Ketika kita melakukan query:

```sql
SELECT * FROM money_example;
```

Hasilnya akan terlihat seperti ini:

```text
Laptop      | $999.99
Mouse       | $25.50
Keyboard    | $75.00
Headphones  | $129.99
```

Secara visual, ini terlihat sangat rapi dan nyaman dibaca. PostgreSQL otomatis menambahkan simbol mata uang dan memastikan format dua angka desimal.

#### Fleksibilitas Input yang Terlihat Menguntungkan

Tipe data `MONEY` memang unggul dalam hal fleksibilitas input:

- Menerima **float literal**
- Menerima **integer literal**
- Menerima **string dengan format mata uang** seperti `'$129.99'`
- Secara otomatis menampilkan hasil dengan **currency symbol**

Bagi pemula, atau saat melihat contoh ini pertama kali, sangat wajar jika berpikir: “Ini ideal untuk menyimpan harga atau nilai transaksi.”

Namun, justru di sinilah jebakan dimulai.

#### Masalah Besar di Balik Tipe Data MONEY

Walaupun terlihat nyaman, tipe data `MONEY` memiliki sejumlah kelemahan mendasar.

Pertama, `MONEY` **bergantung pada locale** (pengaturan regional). Simbol mata uang, pemisah ribuan, dan pemisah desimal bisa berbeda antara satu sistem dan sistem lain. Artinya, hasil query di server A bisa berbeda tampilannya dengan server B, tergantung konfigurasi locale masing-masing.

Kedua, tipe ini **tidak portable**. Jika suatu saat data perlu dipindahkan ke database lain, atau bahkan ke instance PostgreSQL dengan konfigurasi berbeda, Anda bisa menghadapi masalah parsing dan konversi data. Hal ini sangat berisiko untuk sistem yang skalanya berkembang.

Ketiga, `MONEY` **tidak transparan untuk perhitungan numerik**. Operasi matematika pada `MONEY` sering kali membutuhkan casting tambahan, dan perilakunya tidak sejelas tipe numerik seperti `NUMERIC` atau `DECIMAL`. Ini membuat query menjadi lebih rumit dan rawan kesalahan.

Keempat, tipe ini **kurang fleksibel untuk multi-currency**. Jika aplikasi Anda suatu saat perlu menangani lebih dari satu mata uang (misalnya USD, EUR, IDR), `MONEY` justru menjadi penghambat, karena simbol mata uang melekat langsung pada nilai, bukan sebagai data terpisah.

Karena alasan-alasan inilah, meskipun `MONEY` tampak “siap pakai”, praktik terbaik di dunia nyata justru **menghindarinya sama sekali**. Untuk data finansial yang akurat, konsisten, dan aman dalam jangka panjang, tipe seperti `NUMERIC(precision, scale)` jauh lebih direkomendasikan.

### b. Masalah #1: Loss of Precision

Salah satu kelemahan paling mendasar dari tipe data `MONEY` adalah keterbatasannya dalam hal presisi. Secara default, `MONEY` **hanya mendukung dua angka di belakang koma (2 decimal places)**. Batasan ini sering kali tidak terasa di awal, tetapi bisa menimbulkan masalah serius ketika aplikasi mulai menangani perhitungan keuangan yang lebih kompleks.

#### Masalah Utama: Pembulatan Otomatis

Mari kita lihat bagaimana PostgreSQL memperlakukan nilai dengan jumlah desimal yang berbeda saat dikonversi ke tipe `MONEY`.

```sql
SELECT 1.99::MONEY;
-- Result: $1.99 ✅
```

Nilai dengan dua desimal berjalan sempurna.

```sql
SELECT 99.99::MONEY;
-- Result: $99.99 ✅
```

Masih aman, karena sesuai dengan batas presisi `MONEY`.

Namun, perhatikan apa yang terjadi ketika kita memasukkan lebih dari dua angka desimal:

```sql
SELECT 99.876::MONEY;
-- Result: $99.88 ⚠️
```

Angka `99.876` **dibulatkan otomatis** menjadi `99.88`. Digit ketiga di belakang koma hilang tanpa peringatan eksplisit. Inilah yang disebut _loss of precision_ — data berubah nilainya secara diam-diam.

#### Implikasi pada Kasus Nyata

Efek dari pembatasan ini sangat bergantung pada konteks penggunaan.

#### Kasus yang Masih Aman: Mata Uang Standar (2 Desimal)

Untuk mata uang seperti USD yang memang secara umum hanya menggunakan dua angka desimal (cents), `MONEY` terlihat aman.

```sql
SELECT 100.78::MONEY;
-- Result: $100.78
```

Dalam skenario ini, tidak ada informasi yang hilang karena format datanya sesuai dengan batasan `MONEY`.

#### Masalah pada Fractional Cents

Masalah mulai muncul ketika aplikasi membutuhkan presisi lebih tinggi, misalnya untuk perhitungan pajak, bunga, atau pembagian nilai.

```sql
SELECT 100.789::MONEY;
-- Result: $100.79
```

Nilai `100.789` seharusnya disimpan apa adanya, tetapi malah dibulatkan menjadi `100.79`. Sekilas terlihat sepele, tetapi akumulasi pembulatan seperti ini dapat menghasilkan selisih yang signifikan dalam laporan keuangan.

Contoh lain yang lebih berbahaya adalah perhitungan matematis:

```sql
SELECT (100.33 / 3)::MONEY;
-- Result: $33.44
```

Secara matematis, hasil yang benar adalah `33.443333...`. Namun, karena dibatasi dua desimal, nilainya dibulatkan menjadi `33.44`. Dalam sistem finansial, kesalahan kecil seperti ini bisa berdampak besar jika terjadi berulang kali.

#### Masalah Besar pada Cryptocurrency

Untuk aset digital seperti cryptocurrency, tipe `MONEY` praktis tidak bisa digunakan sama sekali.

```sql
SELECT 0.00012345::MONEY;
-- Result: $0.00
```

Hampir seluruh nilai hilang karena semua angka penting berada di luar dua desimal yang didukung. Ini bukan sekadar pembulatan kecil, melainkan **kehilangan nilai yang masif**.

Cryptocurrency seperti Ethereum bahkan menggunakan hingga 18 decimal places, yang membuat tipe `MONEY` sepenuhnya tidak relevan untuk skenario ini.

#### Kesimpulan

Jika aplikasi Anda **hanya** menangani mata uang dengan standar dua desimal dan **tidak pernah** membutuhkan fractional cents, maka secara teknis `MONEY` mungkin masih bisa digunakan. Namun, mengingat risiko kehilangan presisi yang terjadi secara diam-diam, pilihan ini jarang sepadan. Dalam praktik profesional, jauh lebih aman menggunakan tipe data numerik dengan presisi eksplisit agar tidak ada nilai yang “menghilang” tanpa disadari.

### c. Masalah #2: Locale Dependency (LC_MONETARY)

Masalah kedua ini jauh lebih berbahaya dibanding sekadar kehilangan presisi, karena menyentuh **cara data ditampilkan** dan **bagaimana manusia menafsirkan nilainya**. Tipe data `MONEY` sangat bergantung pada pengaturan locale, khususnya `lc_monetary`. Jika setting ini berubah, **representasi nilai uang bisa berubah drastis**, meskipun nilai aslinya sama sekali tidak berubah.

#### Akar Masalah: Formatting Bergantung Locale

PostgreSQL tidak menyimpan informasi mata uang di dalam tipe `MONEY`. Yang disimpan hanyalah **nilai numerik internal**, sedangkan simbol mata uang, pemisah desimal, dan format tampilannya sepenuhnya ditentukan oleh `lc_monetary`.

Mari kita lihat bagaimana masalah ini muncul dalam praktik.

#### Kondisi Awal: Locale US Dollar

Pertama, kita cek locale yang sedang aktif:

```sql
SHOW lc_monetary;
-- Result: en_US.UTF-8
```

Dengan locale Amerika Serikat, ketika kita menampilkan data:

```sql
SELECT * FROM money_example;
```

Hasilnya terlihat wajar dan “benar”:

```text
Laptop      | $999.99
Mouse       | $25.50
Keyboard    | $75.00
```

Simbol dolar (`$`) sesuai dengan asumsi bahwa data tersebut memang disimpan dalam USD.

#### Locale Berubah, Data “Ikut Berubah”

Sekarang bayangkan locale sistem berubah, misalnya ke Inggris:

```sql
SET lc_monetary = 'en_GB.UTF-8';
```

Tanpa mengubah satu pun data di tabel, kita jalankan query yang sama:

```sql
SELECT * FROM money_example;
```

Hasilnya:

```text
Laptop      | £999.99
Mouse       | £25.50
Keyboard    | £75.00
```

Nilai yang sama kini ditampilkan dengan simbol Pound Sterling (`£`).

#### Kenapa Ini Masalah Besar?

Di sinilah letak bahayanya:

- Nilai **sebenarnya** di database adalah `$999.99 USD`
- Nilai yang **terlihat** oleh user adalah `£999.99 GBP`

Tidak ada konversi mata uang sama sekali. PostgreSQL **hanya mengganti simbol**, bukan mengonversi nilainya.

Secara kasar:

- `$999.99 USD` ≈ `£790 GBP`
- Tapi yang ditampilkan: `£999.99 GBP`

Artinya, data terlihat **jauh lebih mahal** dari nilai sebenarnya. Ini bukan sekadar masalah kosmetik, tetapi kesalahan representasi data yang bisa menyesatkan pengambilan keputusan.

#### Ilustrasi Cara Kerja Internal

Secara internal, PostgreSQL hanya menyimpan satu angka:

```text
Stored value (internal): 999.99
```

Cara angka ini ditampilkan sepenuhnya bergantung pada locale:

```text
lc_monetary     →  Display
en_US.UTF-8     →  $999.99
en_GB.UTF-8     →  £999.99
de_DE.UTF-8     →  999,99 €
ja_JP.UTF-8     →  ¥999.99
```

Nilainya **tidak pernah berubah**, yang berubah hanyalah cara menampilkannya. Inilah yang membuat tipe `MONEY` sangat membingungkan dan berpotensi menyesatkan.

#### Skenario Nyata: Multi-Tenant Disaster

Bayangkan sebuah aplikasi multi-tenant:

- Tenant A adalah perusahaan di US dan menyimpan data dalam USD
- Tenant B adalah perusahaan di UK dan mengubah `lc_monetary`

Tenant A menyimpan invoice:

```sql
SET lc_monetary = 'en_US.UTF-8';
INSERT INTO invoices VALUES ('Invoice-001', 1000.00::MONEY);
```

Secara internal, nilai yang disimpan adalah `1000.00` dan tampil sebagai `$1000.00`.

Namun, di waktu lain — misalnya setelah sistem restart atau saat Tenant B aktif:

```sql
SET lc_monetary = 'en_GB.UTF-8';
SELECT * FROM invoices WHERE invoice_id = 'Invoice-001';
```

Hasilnya:

```text
£1000.00
```

Invoice yang seharusnya bernilai **$1000 USD** kini terlihat sebagai **£1000 GBP**. Ini adalah mimpi buruk untuk akuntansi, audit, dan laporan keuangan.

#### Kesimpulan

Tipe data `MONEY` **tidak menyimpan informasi mata uang sama sekali**. Ia hanya menyimpan angka, sementara simbol dan formatnya sepenuhnya dikendalikan oleh locale. Akibatnya, perubahan `lc_monetary` dapat membuat data yang sama terlihat memiliki makna finansial yang berbeda.

Untuk aplikasi yang mendukung multi-currency, multi-tenant, atau berjalan di lingkungan dengan konfigurasi locale yang bisa berubah, kondisi ini **sangat berbahaya**. Inilah salah satu alasan utama kenapa tipe `MONEY` hampir selalu dihindari dalam sistem yang serius dan berskala besar.

### d. SOLUSI #1: Store sebagai INTEGER (Recommended!)

Pendekatan paling umum dan paling aman untuk menyimpan data finansial di database adalah **menyimpan uang sebagai bilangan bulat (INTEGER)**. Ide utamanya sederhana: jangan simpan uang dalam bentuk desimal, tetapi simpan dalam **unit terkecil dari mata uang tersebut**, misalnya cents untuk USD.

Dengan cara ini, kita sepenuhnya menghindari masalah presisi, pembulatan, dan ketergantungan locale yang muncul pada tipe seperti `FLOAT` atau `MONEY`.

#### Konsep Dasar

Alih-alih menyimpan `$100.78` sebagai `100.78`, kita menyimpannya sebagai:

```
10078 cents
```

Artinya:

- `$1.00` → `100`
- `$25.50` → `2550`
- `$100.78` → `10078`

Semua nilai disimpan sebagai **angka bulat**, tanpa koma sama sekali.

#### Pendekatan Implementasi

Contoh implementasi sederhana di PostgreSQL:

```sql
CREATE TABLE products (
    product_name TEXT,
    price_cents INTEGER  -- Disimpan dalam cents, bukan dollars
);
```

Saat memasukkan data, kita langsung memasukkan nilai dalam cents:

```sql
INSERT INTO products VALUES ('Laptop', 10078);
INSERT INTO products VALUES ('Mouse', 2550);
```

Ketika ingin menampilkan data ke user, kita lakukan konversi saat query:

```sql
SELECT
    product_name,
    price_cents,
    price_cents / 100.0 AS price_dollars
FROM products;
```

Hasilnya:

```text
Laptop | 10078 | 100.78
Mouse  | 2550  | 25.50
```

Data disimpan dengan aman sebagai integer, sementara konversi ke format yang ramah manusia dilakukan hanya saat dibutuhkan.

#### Logika Konversi yang Jelas

Konversi dua arah ini sangat sederhana dan konsisten.

Untuk penyimpanan (dollars ke cents):

```sql
SELECT (100.78 * 100)::INT4;
-- Result: 10078
```

Untuk tampilan (cents ke dollars):

```sql
SELECT 10078 / 100.0;
-- Result: 100.78
```

Tidak ada pembulatan tersembunyi, tidak ada kejutan.

#### Menangani Fractional Cents

Pada beberapa kasus, seperti perhitungan pajak, bunga, atau harga berbasis volume, kita mungkin membutuhkan presisi lebih dari sekadar cents.

Solusinya tetap sama: simpan dalam unit yang lebih kecil lagi.

```sql
CREATE TABLE precise_prices (
    product_name TEXT,
    price_millis INTEGER  -- Thousandths of a cent (0.001¢)
);
```

Misalnya:

```sql
INSERT INTO precise_prices VALUES ('Item', 100789);
```

Nilai ini merepresentasikan:

```
100.789 * 1000 = 100789
```

Untuk menampilkan kembali:

```sql
SELECT 100789 / 1000.0;
-- Result: 100.789
```

Pendekatan ini tetap menjaga akurasi penuh tanpa melibatkan angka pecahan saat penyimpanan.

#### Keuntungan Pendekatan INTEGER

#### Performa: Sangat Cepat

Operasi integer adalah operasi paling optimal di CPU:

- Penjumlahan dan pengurangan dilakukan secara native
- Perbandingan sangat cepat
- Indexing pada kolom integer sangat efisien

Ini menjadikan pendekatan ini ideal untuk sistem dengan volume transaksi tinggi.

#### Akurasi: Presisi Sempurna

Dengan integer, tidak ada konsep floating-point error.

```sql
SELECT (10078 + 2550);
-- Result: 12628 (exact)
```

Bandingkan dengan float:

```sql
SELECT (100.78 + 25.50)::FLOAT8;
-- Result: 126.28000000000002
```

Kesalahan kecil seperti ini sering tidak terlihat, tetapi bisa menumpuk dan menimbulkan masalah besar di sistem finansial.

#### Kesederhanaan: Mudah Dipahami dan Diprediksi

Nilai selalu berupa angka bulat:

```
10078 = $100.78
```

Tidak ada ambiguitas, tidak ada kemungkinan menyimpan nilai setengah jalan. Selama konvensinya jelas, logikanya sangat mudah diikuti.

#### API-Friendly dan Terbukti di Dunia Nyata

Banyak payment gateway besar menggunakan pendekatan ini. Salah satu contohnya adalah Stripe, yang selalu menerima amount dalam **smallest currency unit**:

- USD → cents
- EUR → euro cents
- JPY → yen (memang tidak memiliki desimal)

Pendekatan ini terbukti aman dan skalabel.

#### Contoh Nyata: Desain ala Stripe

```sql
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    amount_cents INTEGER,   -- USD cents
    currency CHAR(3),       -- 'USD', 'EUR', 'GBP'
    status TEXT
);
```

Membuat order senilai `$49.99`:

```sql
INSERT INTO orders VALUES
    (1, 100, 4999, 'USD', 'paid');
```

Menampilkan ke user:

```sql
SELECT
    order_id,
    amount_cents / 100.0 AS amount_dollars,
    currency
FROM orders;
```

Hasilnya:

```text
1 | 49.99 | USD
```

#### Trade-offs yang Perlu Dipahami

Pendekatan ini memang bukan tanpa konsekuensi.

#### Kelebihan

- Operasi sangat cepat
- Akurasi sempurna, tanpa kehilangan presisi
- Logika jelas dan konsisten
- Aman untuk integrasi API
- Sulit “tidak sengaja” melakukan perhitungan dengan float

#### Kekurangan

- Perlu konversi saat insert dan saat display
- Developer harus selalu sadar satuan yang digunakan
- Logika aplikasi bertanggung jawab penuh atas konversi
- Kesalahan naming kolom bisa membingungkan

#### Best Practices agar Tetap Aman

Gunakan penamaan kolom yang eksplisit:

```sql
CREATE TABLE invoices (
    amount_cents INTEGER,
    currency_code CHAR(3)
);
```

Sediakan helper function untuk konversi:

```sql
CREATE FUNCTION cents_to_dollars(cents INTEGER)
RETURNS NUMERIC(10,2) AS $$
BEGIN
    RETURN cents / 100.0;
END;
$$ LANGUAGE plpgsql;
```

Gunakan view untuk kebutuhan display:

```sql
CREATE VIEW invoices_display AS
SELECT
    invoice_id,
    cents_to_dollars(amount_cents) AS amount,
    currency_code
FROM invoices;
```

Hindari penamaan yang ambigu:

```sql
CREATE TABLE bad_table (
    price INTEGER  -- Tidak jelas: cents atau dollars?
);
```

Dengan disiplin penamaan dan konvensi yang jelas, pendekatan **INTEGER sebagai penyimpan nilai uang** adalah solusi paling aman, cepat, dan terpercaya untuk sistem finansial modern.

### e. SOLUSI #2: Store sebagai NUMERIC

Solusi kedua yang sangat umum dan juga **sangat aman** untuk menyimpan nilai uang adalah menggunakan tipe data `NUMERIC`. Pendekatan ini menekankan kejelasan dan presisi: nilai uang disimpan sebagai angka desimal dengan **jumlah digit dan jumlah angka di belakang koma yang ditentukan secara eksplisit**.

Berbeda dengan `FLOAT`, tipe `NUMERIC` tidak menggunakan representasi floating-point biner, sehingga **tidak menimbulkan error presisi**. Inilah alasan utama kenapa `NUMERIC` sering menjadi pilihan default di sistem finansial.

#### Konsep Dasar

Dengan `NUMERIC`, kita menentukan dua hal penting:

- **Precision** → total jumlah digit
- **Scale** → jumlah digit di belakang koma

Contoh `NUMERIC(10, 2)` berarti:

- Total maksimal 10 digit
- 2 digit di belakang koma
- Nilai maksimum: `99,999,999.99`

#### Pendekatan Dasar

Contoh penggunaan paling umum untuk mata uang standar:

```sql
CREATE TABLE products (
    product_name TEXT,
    price NUMERIC(10, 2)
);
```

Memasukkan data terasa alami, sama seperti menulis nilai uang sehari-hari:

```sql
INSERT INTO products VALUES
    ('Laptop', 999.99),
    ('Mouse', 25.50);
```

Ketika data di-query:

```sql
SELECT * FROM products;
```

Hasilnya langsung tampil dalam format yang benar:

```text
Laptop | 999.99
Mouse  | 25.50
```

Tidak perlu konversi tambahan untuk keperluan display.

#### Opsi Precision Sesuai Kebutuhan

Salah satu keunggulan utama `NUMERIC` adalah fleksibilitasnya. Kita bisa menyesuaikan precision dan scale sesuai konteks bisnis.

#### Standar Retail (2 Desimal)

```sql
price NUMERIC(10, 2)
```

Cukup untuk e-commerce dan retail, dengan kapasitas hingga hampir 100 juta per transaksi.

#### Transaksi Bernilai Besar

```sql
price NUMERIC(15, 2)
```

Cocok untuk sistem B2B, enterprise, atau perbankan, dengan nilai hingga hampir 10 triliun.

#### Fractional Cents

```sql
price NUMERIC(10, 4)
```

Digunakan untuk kebutuhan seperti perhitungan pajak, bunga, atau instrumen keuangan yang membutuhkan presisi lebih dari dua desimal.

#### Cryptocurrency

```sql
price NUMERIC(30, 18)
```

Mendukung hingga 18 angka di belakang koma, sesuai standar Ethereum. Bitcoin sendiri menggunakan 8 desimal, yang juga terakomodasi dengan mudah.

#### Tanpa Batas (Unbounded)

```sql
price NUMERIC
```

Pendekatan ini memberikan fleksibilitas maksimum, tetapi juga berisiko karena tidak ada validasi bawaan terhadap ukuran dan skala nilai.

#### Contoh Implementasi Lengkap

Berikut contoh tabel transaksi yang lebih realistis:

```sql
CREATE TABLE transactions (
    transaction_id BIGINT PRIMARY KEY,
    amount NUMERIC(10, 2),
    currency CHAR(3),
    description TEXT,
    created_at TIMESTAMP
);
```

Memasukkan berbagai jenis transaksi:

```sql
INSERT INTO transactions VALUES
    (1, 1234.56, 'USD', 'Payment received', NOW()),
    (2, 0.99, 'USD', 'Micro-transaction', NOW()),
    (3, 999999.99, 'USD', 'Large payment', NOW());
```

Melakukan operasi matematika:

```sql
SELECT
    SUM(amount) AS total,
    AVG(amount) AS average,
    MAX(amount) AS largest
FROM transactions;
```

Semua hasil perhitungan di atas **akurat sepenuhnya**, tanpa floating-point error sekecil apa pun.

#### Keunggulan Pendekatan NUMERIC

#### Presisi Sempurna

`NUMERIC` menyimpan angka dalam representasi desimal yang eksak.

```sql
SELECT 100.33::NUMERIC / 3;
```

Hasilnya:

```text
33.443333333333333333...
```

Bandingkan dengan `FLOAT`:

```sql
SELECT 100.33::FLOAT8 / 3;
```

Hasilnya sedikit meleset, meskipun tampak sepele. Dalam sistem finansial, selisih kecil ini tidak bisa ditoleransi.

#### Fleksibel dan Adaptif

Precision bisa disesuaikan kapan pun sesuai kebutuhan domain bisnis, mulai dari retail sederhana hingga aset kripto.

#### Native SQL dan Mudah Digunakan

Nilai langsung siap ditampilkan tanpa konversi tambahan:

```sql
SELECT amount FROM transactions;
```

Ini berbeda dengan pendekatan integer yang membutuhkan pembagian untuk keperluan display.

#### Maksud yang Jelas

Deklarasi seperti:

```sql
price NUMERIC(10, 2)
```

Secara implisit sudah menjelaskan bahwa kolom tersebut adalah angka desimal dengan dua digit di belakang koma. Skema database menjadi lebih mudah dibaca dan dipahami.

#### Trade-off yang Perlu Dipertimbangkan

Pendekatan `NUMERIC` bukan tanpa kompromi.

#### Kelebihan

- Presisi sempurna, sangat krusial untuk data uang
- Representasi alami dan mudah dipahami
- Precision dapat disesuaikan
- Validasi bawaan melalui precision dan scale
- Tidak perlu konversi untuk display

#### Kekurangan

- Performa lebih lambat dibanding INTEGER
- Ukuran penyimpanan bisa lebih besar
- `NUMERIC` tanpa batas berisiko jika tanpa validasi tambahan

Dalam benchmark sederhana, operasi `NUMERIC` bisa beberapa kali lebih lambat dibanding INTEGER. Namun untuk data finansial, **akurasi selalu lebih penting daripada kecepatan mentah**.

#### Best Practices

Gunakan precision yang eksplisit:

```sql
CREATE TABLE prices (
    amount NUMERIC(10, 2) NOT NULL CHECK (amount >= 0)
);
```

Pisahkan nilai dan mata uang:

```sql
CREATE TABLE multi_currency (
    amount NUMERIC(10, 2),
    currency CHAR(3) NOT NULL
);
```

Hindari `NUMERIC` tanpa batas tanpa validasi:

```sql
CREATE TABLE risky (
    amount NUMERIC
);
```

Jika membutuhkan fleksibilitas lebih, tetap berikan batasan:

```sql
CREATE TABLE better (
    amount NUMERIC(15, 2)
    CHECK (amount BETWEEN 0 AND 999999999.99)
);
```

Dengan konfigurasi yang tepat, tipe data `NUMERIC` memberikan keseimbangan terbaik antara **kejelasan, fleksibilitas, dan presisi**, menjadikannya pilihan yang sangat solid untuk penyimpanan data uang di database.

### f. Handling Multiple Currencies

Ketika aplikasi hanya berurusan dengan satu mata uang, desain penyimpanan data uang relatif sederhana. Namun, begitu sistem mulai menangani **lebih dari satu currency**, desain skema database harus dibuat dengan sangat hati-hati agar tidak menimbulkan kebingungan, kesalahan perhitungan, atau representasi data yang menyesatkan.

Masalah utamanya adalah: **bagaimana menyimpan nilai uang tanpa kehilangan konteks mata uangnya**.

#### Prinsip Utama: Pisahkan Nilai dan Mata Uang

Praktik terbaik yang direkomendasikan adalah **menyimpan nilai uang dan kode mata uang dalam kolom terpisah**. Nilai disimpan sebagai angka (INTEGER atau NUMERIC), sedangkan mata uang disimpan sebagai kode standar.

Contoh desain yang direkomendasikan:

```sql
CREATE TABLE transactions (
    transaction_id BIGINT PRIMARY KEY,
    amount NUMERIC(15, 2),     -- Bisa juga INTEGER (dalam unit terkecil)
    currency_code CHAR(3),     -- ISO 4217: 'USD', 'EUR', 'GBP'
    description TEXT
);
```

Dengan skema ini, setiap baris data memiliki konteks mata uang yang jelas dan tidak ambigu.

#### Contoh Penggunaan dengan Berbagai Mata Uang

Memasukkan data dari berbagai negara:

```sql
INSERT INTO transactions VALUES
    (1, 100.00, 'USD', 'US payment'),
    (2, 85.50, 'EUR', 'European payment'),
    (3, 75.99, 'GBP', 'UK payment');
```

Ketika melakukan agregasi, kita bisa mengelompokkan berdasarkan mata uang:

```sql
SELECT
    SUM(amount) AS total,
    currency_code
FROM transactions
GROUP BY currency_code;
```

Hasilnya:

```text
100.00 | USD
85.50  | EUR
75.99  | GBP
```

Nilai tidak pernah dicampur antar currency, sehingga hasilnya tetap masuk akal secara finansial.

#### Rekomendasi Desain Skema Berdasarkan Kebutuhan

Tidak semua sistem membutuhkan fleksibilitas yang sama. Desain skema sebaiknya disesuaikan dengan kebutuhan nyata aplikasi.

#### Kasus 1: Selalu Satu Mata Uang

Jika aplikasi **dipastikan** hanya akan menggunakan satu mata uang, misalnya USD, maka menyimpan kolom mata uang justru menjadi redundan.

```sql
CREATE TABLE us_only (
    amount NUMERIC(10, 2)
);
```

Dalam skema ini, mata uang diasumsikan secara implisit. Pendekatan ini aman selama asumsi tersebut tidak pernah berubah.

#### Kasus 2: Mendukung Banyak Mata Uang

Jika aplikasi mendukung lebih dari satu currency, maka kolom mata uang **wajib ada**.

```sql
CREATE TABLE multi_currency (
    amount NUMERIC(10, 2) NOT NULL,
    currency_code CHAR(3) NOT NULL,
    CHECK (currency_code IN ('USD', 'EUR', 'GBP', 'JPY'))
);
```

Constraint ini membantu mencegah nilai mata uang yang tidak valid masuk ke database.

#### Kasus 3: Validasi dengan Foreign Key

Untuk sistem yang lebih kompleks dan terstruktur, sebaiknya mata uang direferensikan ke tabel khusus.

```sql
CREATE TABLE currencies (
    code CHAR(3) PRIMARY KEY,
    name TEXT,
    decimal_places SMALLINT,
    symbol TEXT
);
```

Contoh data referensi:

```sql
INSERT INTO currencies VALUES
    ('USD', 'US Dollar', 2, '$'),
    ('EUR', 'Euro', 2, '€'),
    ('JPY', 'Japanese Yen', 0, '¥'),
    ('BTC', 'Bitcoin', 8, '₿');
```

Kemudian, tabel transaksi bisa mengacu ke tabel ini:

```sql
CREATE TABLE transactions (
    amount NUMERIC(15, 8),
    currency_code CHAR(3) REFERENCES currencies(code)
);
```

Dengan pendekatan ini, validasi mata uang terpusat dan konsisten, serta sistem siap menangani perbedaan jumlah desimal antar currency.

#### Pendekatan INTEGER untuk Multi-Currency

Pendekatan INTEGER tetap bisa digunakan, tetapi logikanya menjadi lebih kompleks karena setiap mata uang memiliki **unit terkecil yang berbeda**.

```sql
CREATE TABLE transactions (
    amount_smallest_unit BIGINT,
    currency_code CHAR(3)
);
```

Makna dari `amount_smallest_unit` bergantung pada `currency_code`:

- USD → cents (bagi 100 untuk display)
- JPY → yen (tidak perlu dibagi)
- BTC → satoshi (bagi 100.000.000)

Artinya, logika konversi harus ditangani secara eksplisit di level aplikasi atau query.

#### Peringatan Keras tentang Tipe MONEY

Menggunakan tipe `MONEY` untuk skenario multi-currency adalah kesalahan desain yang serius.

```sql
CREATE TABLE bad_multi (
    amount MONEY
);
```

Tipe ini tidak menyimpan informasi mata uang sama sekali dan sepenuhnya bergantung pada `lc_monetary`. Dalam konteks multi-currency, ini adalah **bencana yang tinggal menunggu waktu**.

#### Kesimpulan

Untuk menangani multiple currencies dengan aman dan jelas:

- Selalu pisahkan nilai dan mata uang
- Gunakan `NUMERIC` atau `INTEGER`, bukan `MONEY`
- Simpan kode mata uang sesuai standar (ISO 4217)
- Tambahkan validasi melalui CHECK atau foreign key

Dengan desain seperti ini, data keuangan tetap konsisten, mudah dipahami, dan aman untuk dikembangkan dalam jangka panjang.

## 3. Hubungan Antar Konsep

**Decision tree untuk storing money:**

```
Perlu store money di database?
│
├─ Single currency selamanya (e.g., USD only)?
│  ├─ Ya, dan TIDAK perlu fractional cents
│  │  └─ Option 1: INTEGER (cents) ⚡ FASTEST
│  │     atau
│  │     Option 2: NUMERIC(10, 2) ✅ SAFER
│  │
│  └─ Perlu fractional cents atau crypto?
│     └─ NUMERIC(10, 4) atau NUMERIC(30, 18)
│
└─ Multiple currencies?
   └─ NUMERIC(15, 2) + currency_code column
      atau
      INTEGER + currency_code (dengan conversion logic)

❌ JANGAN PAKAI: MONEY type!
```

**Comparison matrix:**

```
╔══════════════╦═══════════╦═══════════╦══════════════╦═══════════╗
║ Approach     ║ Speed     ║ Precision ║ Multi-Curr   ║ Simplicity║
╠══════════════╬═══════════╬═══════════╬══════════════╬═══════════╣
║ MONEY        ║ Fast      ║ 2 decimal ║ ❌ Confusing ║ ❌ Mislead║
║ INTEGER      ║ ⚡⚡⚡       ║ Perfect   ║ ⚠️ Complex   ║ ⚠️ Conv.  ║
║ NUMERIC      ║ Slower    ║ Perfect   ║ ✅ Easy      ║ ✅ Natural║
╚══════════════╩═══════════╩═══════════╩══════════════╩═══════════╝
```

**Progression dari video series:**

```
Video sebelumnya: INTEGER vs NUMERIC vs FLOAT
├─ INTEGER: Fast, whole only
├─ NUMERIC: Precise, slow, fractional
└─ FLOAT: Fast, approximate, fractional

Video ini: Applied to MONEY
├─ ❌ FLOAT: Never untuk money (approximate)
├─ ❌ MONEY: Looks good, has serious issues
├─ ✅ INTEGER: Best performance (Stripe approach)
└─ ✅ NUMERIC: Best clarity (most common)
```

## 4. Kesimpulan

PostgreSQL memiliki tipe data `MONEY`, tetapi **sangat tidak direkomendasikan untuk production use** karena dua masalah serius:

1. **Limited precision:** Hanya 2 decimal places (tidak cocok untuk fractional cents atau cryptocurrency)
2. **Locale dependency:** Display bergantung pada `lc_monetary` setting yang bisa berubah, menyebabkan data terlihat misleading (tidak ada currency conversion sebenarnya)

**Dua solusi recommended:**

### **SOLUSI #1: INTEGER (Smallest Unit)**

**Cara:** Store dalam unit terkecil (cents untuk USD) sebagai INTEGER

**Pros:**

- ⚡⚡⚡ **Blazing fast** - fastest option available
- ✅ **Perfect precision** - no rounding errors
- 🎯 **Type-safe** - can't accidentally use float
- 📦 **API-friendly** - Stripe, PayPal standard

**Cons:**

- 🔄 **Conversion overhead** - must multiply/divide
- 🧠 **Mental overhead** - "10078 means $100.78"
- ⚠️ **Risk** - developer bisa lupa convert untuk display

**Best for:**

- High-performance payment systems
- API integrations (Stripe-style)
- High-volume transaction processing
- Single currency with clear conversion logic

**Example:**

```sql
CREATE TABLE orders (
    amount_cents INTEGER NOT NULL CHECK (amount_cents >= 0),
    currency CHAR(3) DEFAULT 'USD'
);
-- $100.78 stored as: 10078
```

### **SOLUSI #2: NUMERIC (dengan Precision/Scale)**

**Cara:** Store sebagai `NUMERIC(10, 2)` atau sesuai kebutuhan

**Pros:**

- ✅ **Perfect precision** - no loss of accuracy
- 🎨 **Natural representation** - 100.78 means $100.78
- 🔧 **Flexible** - adjust precision per need
- ✅ **Clear** - self-documenting, easy to understand

**Cons:**

- 🐌 **Slower** - ~2-3x slower than INTEGER
- 💾 **Larger storage** - variable size, typically 6-8 bytes

**Best for:**

- Multi-currency applications
- Standard e-commerce (most common choice)
- Financial systems yang butuh clarity
- Cryptocurrency (dengan precision tinggi)
- Tax calculations (fractional cents)

**Example:**

```sql
CREATE TABLE orders (
    amount NUMERIC(10, 2) NOT NULL CHECK (amount >= 0),
    currency CHAR(3) DEFAULT 'USD'
);
-- $100.78 stored as: 100.78
```

### **Multi-Currency Handling**

**Untuk single currency forever:** Tidak perlu currency column
**Untuk multi-currency:** **ALWAYS** tambahkan `currency_code CHAR(3)` column

```sql
CREATE TABLE transactions (
    amount NUMERIC(15, 2),
    currency_code CHAR(3) NOT NULL  -- 'USD', 'EUR', 'GBP'
);
```

### **Final Recommendations**

**Default choice (90% of cases):**

```sql
-- Safe, clear, flexible
CREATE TABLE products (
    price NUMERIC(10, 2) NOT NULL CHECK (price > 0)
);
```

**Performance-critical (Fintech):**

```sql
-- Stripe-style
CREATE TABLE transactions (
    amount_cents INTEGER NOT NULL CHECK (amount_cents >= 0),
    currency CHAR(3) NOT NULL
);
```

**Optional use of MONEY:**

- ✅ OK untuk casting NUMERIC→MONEY **hanya untuk display**
- ❌ NOT OK untuk storage

**Golden rules:**

1. ❌ **NEVER use FLOAT for money** (precision loss)
2. ❌ **AVOID MONEY type for storage** (locale issues)
3. ✅ **Use NUMERIC** (most common, safe default)
4. ✅ **Use INTEGER** (if performance critical)
5. ✅ **Always include currency_code** (if multi-currency)

**Quote dari video:**

> "I'm going to beg you to not use it [MONEY] to store money in the database... I would not rely on money at all for storage. Store it as an integer or store it as a numeric."

Pilihan antara INTEGER dan NUMERIC adalah trade-off antara **performance vs clarity**. Untuk most applications, **NUMERIC(10, 2) adalah safe default**. Untuk high-performance systems seperti payment processors, **INTEGER approach (Stripe-style) adalah pilihan terbaik**.
