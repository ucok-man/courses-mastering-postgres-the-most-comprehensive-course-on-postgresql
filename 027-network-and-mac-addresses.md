# 📝 Network Addresses dan MAC Addresses di PostgreSQL

## 1. Ringkasan Singkat

Materi ini membahas cara menyimpan **network addresses** (alamat IP) dan **MAC addresses** di PostgreSQL menggunakan tipe data khusus. Meskipun tidak terlalu umum, fitur ini sangat berguna ketika dibutuhkan karena lebih efisien dalam hal storage dan query performance dibandingkan menggunakan representasi text biasa. PostgreSQL menyediakan tiga tipe data khusus: `INET` (untuk alamat IP individual), `CIDR` (untuk network ranges), dan `MACADDR`/`MACADDR8` (untuk MAC addresses).

## 2. Konsep Utama

### a. Tipe Data INET

- **INET** digunakan untuk menyimpan alamat IP (baik IPv4 maupun IPv6)
- Dapat menyimpan alamat host saja atau dengan subnet mask
- Lebih efisien dalam storage dan query dibandingkan text

**Contoh pembuatan tabel:**

```sql
CREATE TABLE inet_examples (
    ip_address INET
);
```

**Contoh data yang bisa disimpan:**

```sql
INSERT INTO inet_examples (ip_address) VALUES
    ('192.168.1.1'),              -- Host address saja (IPv4)
    ('192.168.1.0/24'),           -- Dengan subnet mask
    ('2001:0db8::1'),             -- IPv6 address
    ('10.0.0.0/8');               -- IPv4 dengan subnet

SELECT * FROM inet_examples;
```

**Efisiensi Storage:**

```sql
-- Sebagai text: 40 bytes
SELECT pg_column_size(ip_address::TEXT) FROM inet_examples;

-- Sebagai INET: 22 bytes (hampir 50% lebih kecil!)
SELECT pg_column_size(ip_address::INET) FROM inet_examples;
```

### b. Fungsi-Fungsi untuk INET

PostgreSQL menyediakan berbagai fungsi untuk memanipulasi data INET:

**Fungsi-fungsi umum:**

```sql
SELECT
    host(ip_address),           -- Mengambil host saja (tanpa mask)
    masklen(ip_address),        -- Panjang subnet mask
    network(ip_address),        -- Network address
    abbrev(ip_address)          -- Versi abbreviated dari IP
FROM inet_examples;
```

**Hasil contoh:**

```
host          | masklen | network       | abbrev
--------------+---------+---------------+--------------
192.168.1.1   | 32      | 192.168.1.1/32| 192.168.1.1
192.168.1.0   | 24      | 192.168.1.0/24| 192.168.1/24
2001:db8::1   | 128     | 2001:db8::1/128| 2001:db8::1
```

**Keuntungan:**

- Tidak perlu parsing text untuk mendapatkan bagian-bagian dari IP address
- Query lebih cepat dan efisien
- Bisa melakukan operasi range checking dengan mudah

### c. Tipe Data CIDR

- **CIDR** (Classless Inter-Domain Routing) digunakan untuk menyimpan **network ranges**
- Lebih cocok untuk representasi subnet/network daripada individual host
- Sintaks sama seperti INET tetapi fokus pada network

**Contoh pembuatan dan penggunaan:**

```sql
CREATE TABLE cidr_examples (
    network CIDR
);

INSERT INTO cidr_examples (network) VALUES
    ('192.168.1.0/24'),         -- IPv4 network
    ('10.0.0.0/8'),             -- Larger IPv4 network
    ('2001:db8::/32');          -- IPv6 network

SELECT * FROM cidr_examples;
```

**Perbedaan INET vs CIDR:**

| Aspek        | INET                       | CIDR                        |
| ------------ | -------------------------- | --------------------------- |
| **Fokus**    | Individual IP addresses    | Network ranges              |
| **Use case** | Menyimpan IP host tertentu | Menyimpan subnet/network    |
| **Storage**  | Non-text, compact          | Non-text, compact           |
| **Query**    | Cek apakah IP dalam range  | Operasi pada network ranges |

**Kapan menggunakan CIDR:**

- Ketika perlu menyimpan dan query network ranges
- Untuk operasi "apakah IP X ada dalam network Y?"
- Untuk manajemen subnet

### d. Tipe Data MAC Addresses

- PostgreSQL menyediakan dua tipe untuk MAC addresses: `MACADDR` (6 bytes) dan `MACADDR8` (8 bytes)
- Berguna untuk aplikasi network management atau IoT
- Lebih compact dibanding text representation

**Dua versi MAC address:**

```sql
-- MAC Address 6 bytes (standar)
SELECT '08:00:2b:01:02:03'::MACADDR;

-- MAC Address 8 bytes (EUI-64 format)
SELECT '08:00:2b:01:02:03:04:05'::MACADDR8;
```

**Perbandingan ukuran:**

```sql
-- Sebagai text: 21 bytes
SELECT pg_column_size('08:00:2b:01:02:03'::TEXT);

-- Sebagai MACADDR: 6 bytes
SELECT pg_column_size('08:00:2b:01:02:03'::MACADDR);

-- Sebagai MACADDR8: 8 bytes
SELECT pg_column_size('08:00:2b:01:02:03:04:05'::MACADDR8);
```

**Konversi antara tipe:**

```sql
-- MACADDR (6 bytes) bisa di-cast ke MACADDR8 (8 bytes)
SELECT '08:00:2b:01:02:03'::MACADDR::MACADDR8;

-- MACADDR8 bisa menyimpan versi 6 atau 8 bytes
-- Jika punya 8-byte MAC, HARUS menggunakan MACADDR8
```

**Kapan menggunakan masing-masing:**

- `MACADDR` → untuk MAC address standar 6 bytes
- `MACADDR8` → untuk MAC address 8 bytes (EUI-64) atau jika ingin fleksibilitas

### e. Prinsip Pemilihan Tipe Data

**Prinsip fundamental:** Pilih tipe data yang **paling akurat merepresentasikan realitas**

```
Realitas:        Tipe Data yang Benar:
----------------|-----------------------
IP Address      | INET (bukan TEXT)
Network Range   | CIDR (bukan TEXT)
MAC Address     | MACADDR/MACADDR8 (bukan TEXT)
```

**Alasan:**

1. **Storage lebih efisien** → Hingga 50% lebih kecil
2. **Query lebih cepat** → Tidak perlu parsing text
3. **Operasi built-in** → Fungsi-fungsi khusus tersedia
4. **Type safety** → Database memvalidasi format

**Contoh perbandingan:**

```sql
-- ❌ Cara yang kurang efisien
CREATE TABLE devices_bad (
    ip_address TEXT,
    mac_address TEXT
);

-- ✅ Cara yang lebih efisien
CREATE TABLE devices_good (
    ip_address INET,
    mac_address MACADDR
);
```

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
