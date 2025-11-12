# 📝 Binary Data di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang penyimpanan binary data (data mentah dalam bentuk bytes) di PostgreSQL menggunakan tipe data `BYTEA`. Berbeda dengan TEXT yang menyimpan string dengan encoding dan collation, BYTEA menyimpan raw bytes tanpa interpretasi karakter. Video menjelaskan kapan sebaiknya menggunakan binary data, use cases yang valid (seperti checksums dan hashes), dan perbandingan ukuran storage antara berbagai representasi hash (MD5 sebagai text, BYTEA, dan UUID).

## 2. Konsep Utama

### a. Tipe Data BYTEA

BYTEA (byte array) adalah tipe data PostgreSQL untuk menyimpan binary data mentah.

**Karakteristik BYTEA:**

- Variable-length column (ukuran berubah sesuai data)
- Maksimum: 1 GB per value (dengan protocol limitation 2 GiB - header saat SELECT)
- Tidak ada encoding atau collation (raw bytes)
- TOAST-enabled untuk data besar
- Comparison berdasarkan byte sequence, bukan karakter
- Overhead: 1 byte untuk data < 127 bytes, 4 bytes untuk data ≥ 127 bytes (on-disk storage)

```sql
-- Membuat table dengan BYTEA column
CREATE TABLE bytea_example (
    file_name TEXT,
    data BYTEA
);

-- Insert data sebagai hex (format: \x + hex digits)
INSERT INTO bytea_example VALUES (
    'hello.txt',
    '\x48656c6c6f20576f726c64'  -- "Hello World" dalam hex
);

-- View data
SELECT * FROM bytea_example;
-- Output akan show binary data atau hex tergantung client
```

**Hex Representation:**

```
String: "Hello World"
Bytes:  48 65 6c 6c 6f 20 57 6f 72 6c 64
Hex:    \x48656c6c6f20576f726c64

H = 0x48 = 72 (decimal)
e = 0x65 = 101
l = 0x6c = 108
...
```

### b. BYTEA vs TEXT

Perbedaan fundamental antara binary data dan text data:

| Aspek          | TEXT               | BYTEA             |
| -------------- | ------------------ | ----------------- |
| **Content**    | Characters         | Raw bytes         |
| **Encoding**   | UTF-8, Latin1, dll | No encoding       |
| **Collation**  | en_US.UTF-8, dll   | No collation      |
| **Comparison** | Character rules    | Byte-by-byte      |
| **Use case**   | Human-readable     | Machine data      |
| **Overhead**   | 1-4 bytes varlena  | 1-4 bytes varlena |

```sql
-- TEXT: Menyimpan dengan encoding dan collation
CREATE TABLE text_table (
    content TEXT  -- Stored with UTF-8 encoding
);

-- BYTEA: Menyimpan raw bytes
CREATE TABLE binary_table (
    content BYTEA  -- Stored as-is, no interpretation
);

-- Comparison berbeda:
-- TEXT: 'abc' vs 'ABC' tergantung collation
-- BYTEA: 0x616263 vs 0x414243 byte-by-byte exact match
```

### c. BYTEA Output Formats

PostgreSQL mendukung dua format output untuk BYTEA:

#### 1. Hex Format (Default & Recommended)

```sql
-- Melihat format output saat ini
SHOW bytea_output;
-- Output: hex (default)

-- Data ditampilkan dalam hexadecimal
SELECT data FROM bytea_example;
-- Output: \x48656c6c6f20576f726c64
```

**Keuntungan Hex:**

- ✅ Tidak ambiguous (setiap byte = 2 hex digits)
- ✅ Standard format
- ✅ Tool-friendly
- ✅ Tidak ada data loss

#### 2. Escape Format (Legacy)

```sql
-- Mengubah ke escape format
SET bytea_output = 'escape';

SELECT data FROM bytea_example;
-- Output: Hello World (jika bytes adalah valid ASCII)
-- Atau: \000\001\002... (untuk non-ASCII bytes)
```

**Masalah Escape:**

- ⚠️ Tidak semua bytes bisa direpresentasikan sebagai character
- ⚠️ Output bisa membingungkan dengan escape sequences
- ⚠️ Legacy format, tidak recommended

**Perbandingan:**

```sql
-- Data: Binary image header
-- Hex format:    \x89504e470d0a1a0a  ✅ Jelas
-- Escape format: \211PNG\r\n\032\n  ⚠️ Campuran readable/escape
```

### d. TOAST untuk Binary Data

Sama seperti TEXT, BYTEA juga menggunakan TOAST untuk data besar.

**Cara Kerja TOAST dengan BYTEA:**

```sql
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    name TEXT,
    content BYTEA  -- Bisa sampai 1GB
);

-- Insert file kecil (10 KB)
INSERT INTO files VALUES (1, 'small.jpg', '\x...');
-- Disimpan inline di row

-- Insert file besar (500 KB)
INSERT INTO files VALUES (2, 'large.jpg', '\x...');
-- Otomatis di-TOAST:
-- - Row hanya simpan pointer
-- - Actual data di TOAST table
-- - Row tetap compact ✅

-- Query tanpa content column
SELECT id, name FROM files;
-- Super cepat! Tidak perlu akses TOAST table

-- Query dengan content column
SELECT * FROM files WHERE id = 2;
-- Fetch dari TOAST table saat dibutuhkan
```

### e. Kapan (Tidak) Menggunakan BYTEA untuk Files

Video memberikan opini kuat tentang penyimpanan files di database.

#### ❌ TIDAK Recommended: Large Files in Database

```sql
-- ❌ JANGAN: Store large files di database
CREATE TABLE user_uploads (
    user_id INTEGER,
    file_name TEXT,
    file_data BYTEA  -- 50MB per file
);

-- Masalah:
-- 1. Database bengkak
-- 2. Backup lambat
-- 3. Replication overhead
-- 4. Memory pressure saat query
-- 5. Tidak bisa pakai CDN
```

#### ✅ Recommended: File Storage + Pointer

```sql
-- ✅ LAKUKAN: Store file di S3/R2/Tigris, pointer di DB
CREATE TABLE user_uploads (
    user_id INTEGER,
    file_name TEXT,
    file_url TEXT,  -- https://s3.amazonaws.com/bucket/file.jpg
    file_size INTEGER,
    uploaded_at TIMESTAMP
);

-- Keuntungan:
-- 1. Database tetap kecil
-- 2. Bisa pakai CDN untuk delivery
-- 3. Backup cepat
-- 4. Scale independently
```

**Trade-off:**

```
Database Storage
├─ Pro: Strong consistency, ACID guarantees
├─ Pro: Single source of truth
└─ Con: Not designed for large binary blobs

File Storage (S3, etc)
├─ Pro: Optimized untuk files
├─ Pro: CDN integration
├─ Pro: Cheaper per GB
└─ Con: Eventually consistent, bisa out-of-sync dengan DB
```

#### ⚖️ Maybe OK: Very Small Files

```sql
-- Mungkin OK untuk files sangat kecil (< 100KB)
CREATE TABLE user_profiles (
    user_id INTEGER,
    avatar BYTEA  -- Small thumbnail (50KB)
);

-- Keuntungan:
-- ✅ Strong consistency (avatar + user data dalam 1 transaction)
-- ✅ Tidak perlu manage S3 credentials
-- ✅ Atomicity guaranteed

-- Pertimbangan:
-- - Apakah benar-benar butuh consistency?
-- - Apakah file akan grow?
-- - Apakah butuh CDN?
```

**Managed Providers (Supabase, Zeta):**

```sql
-- Beberapa provider handle ini otomatis
-- File > threshold (misal 5MB) → auto-moved ke S3 + CDN
-- User tidak perlu manage
CREATE TABLE files (
    content BYTEA  -- Provider handle TOAST → S3 transition
);
```

### f. Use Case Valid: Checksums dan Hashes

Binary data sangat cocok untuk menyimpan hashes dan checksums.

#### MD5 Hash (⚠️ Not Secure!)

```sql
-- Generate MD5 hash
SELECT MD5('Hello World');
-- Output: 3e25960a79dbc69b674cd4ec67a72c62 (TEXT)

-- Cek tipe data
SELECT pg_typeof(MD5('Hello World'));
-- Output: text

-- ⚠️ WARNING: MD5 TIDAK SECURE!
-- JANGAN gunakan untuk:
-- - Password hashing
-- - Cryptographic security
-- - Anything security-critical

-- ✅ OK untuk:
-- - Checksums (detect data corruption)
-- - Non-cryptographic hashing
-- - Cache keys
-- - Deduplication
```

**MD5 Security Warning:**

```
MD5 Status: CRYPTOGRAPHICALLY BROKEN ❌

Vulnerable to:
- Collision attacks
- Preimage attacks
- Used for decades, fully analyzed

Use instead for security:
- SHA-256 (or better)
- bcrypt (for passwords)
- argon2 (for passwords)

Still OK for:
- File checksums (detect corruption, not tampering)
- Fast hashing where security doesn't matter
```

#### SHA-256 (Secure Alternative)

```sql
-- SHA-256 hash (requires pgcrypto extension)
SELECT digest('Hello World', 'sha256');
-- Output: \xa591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e

-- Cek tipe data
SELECT pg_typeof(digest('Hello World', 'sha256'));
-- Output: bytea (berbeda dengan MD5!)

-- ✅ SHA-256 adalah secure
-- Gunakan untuk security-critical operations
```

**Quirk: MD5 vs SHA-256 Return Types**

```sql
-- MD5 returns TEXT
MD5('data') → '3e25960...' (TEXT)

-- SHA-256 returns BYTEA
digest('data', 'sha256') → \xa591... (BYTEA)

-- Inconsistent, tapi begitulah 🤷
```

### g. Converting MD5 to BYTEA

Untuk performa dan storage efficiency, convert MD5 text ke BYTEA.

```sql
-- MD5 sebagai text (default)
SELECT MD5('Hello World');
-- Output: 3e25960a79dbc69b674cd4ec67a72c62 (32 chars TEXT)

-- Convert ke BYTEA dengan decode
SELECT decode(MD5('Hello World'), 'hex');
-- Output: \x3e25960a79dbc69b674cd4ec67a72c62 (16 bytes BYTEA)

-- Perbandingan ukuran (VERIFIED!) ✅
SELECT
    pg_column_size(MD5('Hello World')) AS md5_text_size,
    pg_column_size(decode(MD5('Hello World'), 'hex')) AS md5_bytea_size;
-- Output:
-- md5_text_size: 36 bytes (32 chars + 4 bytes TEXT varlena overhead)
-- md5_bytea_size: 20 bytes (16 bytes data + 4 bytes BYTEA varlena overhead)
```

**Why Convert?**

1. **Ukuran lebih kecil**: 20 bytes vs 36 bytes (44% saving!)
2. **Comparison lebih cepat**: Byte comparison vs character comparison
3. **Tidak ada collation overhead**: Raw bytes
4. **Predictable size**: Fixed 16-byte data portion

**Storage Overhead Explained:**

```
PostgreSQL pg_column_size() measures TOTAL storage including overhead:

MD5 as TEXT (32 hex chars):
- Data: 32 bytes
- Varlena overhead: 4 bytes
- Total: 36 bytes

MD5 as BYTEA (16 bytes binary):
- Data: 16 bytes
- Varlena overhead: 4 bytes
- Total: 20 bytes

Note: On-disk overhead varies (1 byte for <127 bytes, 4 bytes for ≥127 bytes)
but pg_column_size() shows the actual storage size which includes padding.
```

**Process:**

```
'Hello World'
    ↓ MD5()
'3e25960a79dbc69b674cd4ec67a72c62' (32 hex chars as TEXT)
    ↓ decode(..., 'hex')
\x3e25960a79dbc69b674cd4ec67a72c62 (16 bytes as BYTEA)
```

### h. UUID: Best Balance of Ergonomics & Efficiency

PostgreSQL punya tipe UUID yang optimal untuk menyimpan hash identifiers.

```sql
-- Cast MD5 ke UUID
SELECT MD5('Hello World')::UUID;
-- Output: 3e25960a-79db-c69b-674c-d4ec67a72c62

-- Format UUID dengan dashes untuk readability!

-- Perbandingan ukuran storage (VERIFIED!) ✅
SELECT
    pg_column_size(MD5('Hello World')) AS text_size,
    pg_column_size(decode(MD5('Hello World'), 'hex')) AS bytea_size,
    pg_column_size(MD5('Hello World')::UUID) AS uuid_size;
-- Output:
-- text_size: 36 bytes
-- bytea_size: 20 bytes
-- uuid_size: 16 bytes ✅ (SMALLEST!)
```

**UUID Format:**

```
MD5 output:     3e25960a79dbc69b674cd4ec67a72c62
UUID format:    3e25960a-79db-c69b-674c-d4ec67a72c62
                └──┬───┘ └─┬┘ └─┬┘ └─┬┘ └────┬─────┘
                   │       │    │    │       │
                8 digits   4    4    4    12 digits

Internal Storage: 16 bytes (128-bit fixed-size, NO varlena overhead!)
Display: Formatted dengan dashes (human-readable!)
```

**Why UUID is The Best Choice:**

```sql
CREATE TABLE file_checksums (
    file_id INTEGER,
    checksum_text TEXT,           -- 36 bytes
    checksum_bytea BYTEA,         -- 20 bytes
    checksum_uuid UUID            -- 16 bytes ✅ BEST!
);

-- UUID advantages:
-- 1. SMALLEST storage (16 bytes, no varlena overhead!)
-- 2. Native PostgreSQL type (better optimizer support)
-- 3. Fixed-size (predictable, excellent for indexing)
-- 4. Readable display (formatted dengan dashes)
-- 5. Standard format (RFC 4122, RFC 9562)
-- 6. Can generate with gen_random_uuid()
-- 7. Better ecosystem/tooling support
-- 8. Fast comparison (fixed 16-byte comparison)

-- BYTEA trade-offs:
-- + Maximum flexibility (any hash algorithm, any binary data)
-- - Larger storage (20 bytes dengan varlena overhead)
-- - Variable-length complications

-- TEXT trade-offs:
-- + Human-readable without conversion
-- - LARGEST storage (36 bytes)
-- - Slower comparison (collation overhead)
```

**Key Insight: UUID is Fixed-Size Type!**

```
UUID adalah tipe fixed-size yang disimpan sebagai 16 bytes TANPA varlena overhead!
Inilah mengapa UUID lebih kecil dari BYTEA meskipun keduanya menyimpan 16 bytes data.

UUID:  16 bytes data + 0 bytes overhead = 16 bytes total ✅
BYTEA: 16 bytes data + 4 bytes overhead = 20 bytes total
TEXT:  32 bytes data + 4 bytes overhead = 36 bytes total
```

### i. Practical Use Case: Content Deduplication

Menggunakan hash untuk mendeteksi duplicate content.

```sql
-- Table untuk store files dengan deduplication
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    file_name TEXT,
    content BYTEA,
    content_hash UUID GENERATED ALWAYS AS (MD5(content)::UUID) STORED
);

-- Index pada hash untuk fast lookup
CREATE INDEX idx_files_hash ON files(content_hash);

-- Insert file
INSERT INTO files (file_name, content)
VALUES ('doc1.txt', '\x48656c6c6f');

-- Check for duplicates sebelum insert
WITH new_file AS (
    SELECT '\x48656c6c6f'::BYTEA AS content
)
SELECT f.*
FROM files f, new_file nf
WHERE f.content_hash = MD5(nf.content)::UUID;

-- Jika ada result → file duplicate, jangan insert!
```

**Flow Deduplication:**

```
User uploads file
    ↓
Calculate MD5 hash
    ↓
Check if hash exists in DB
    ↓
├─ Exists → Show "File already uploaded"
└─ Not exists → Insert new file
```

## 3. Hubungan Antar Konsep

### Storage Efficiency Hierarchy (VERIFIED!):

```
Most Efficient
    ↑
UUID (MD5 cast)          16 bytes  ✅ SMALLEST! (fixed-size, no overhead)
    ↑
BYTEA (decoded hex)      20 bytes  ✅ Flexible (varlena overhead)
    ↑
TEXT (hex string)        36 bytes  ⚠️ WASTEFUL (readable but big)
    ↑
Least Efficient
```

### Data Type Decision Tree:

```
Apa yang mau disimpan?

├─ Human-readable text
│   └─ TEXT (dengan encoding & collation)
│
├─ Machine data / bytes
│   ├─ Small (< 1KB): BYTEA ✅
│   ├─ Medium (1KB-1MB): BYTEA dengan consideration
│   └─ Large (> 1MB): File storage + pointer ✅
│
├─ Hash / Checksum (MD5, SHA, etc)
│   ├─ Best choice: UUID ✅ (16 bytes, native type)
│   ├─ Flexible: BYTEA ✅ (20 bytes, works with any hash)
│   └─ Human-readable: TEXT (36 bytes, wasteful)
│
└─ Files
    ├─ Very small (< 100KB): Maybe BYTEA
    └─ Anything larger: S3/R2 + URL pointer ✅
```

### Why UUID Wins for Hashes:

```
UUID advantages over BYTEA for MD5/hash storage:

1. Smaller: 16 bytes vs 20 bytes (20% space saving)
2. Faster: Fixed-size comparison vs variable-length
3. Indexing: Better index performance
4. Standard: RFC-compliant UUID type
5. Tooling: Better ecosystem support
6. Readable: Formatted output with dashes

Use BYTEA when:
- Need flexibility (SHA-256, SHA-512, custom hashes)
- Storing non-hash binary data
- Hash size varies

Use UUID when:
- Storing 128-bit hashes (MD5, etc)
- Want smallest storage
- Want best performance
```

### TOAST Usage:

```
TEXT column
    └─ Large text (> ~2KB) → TOAST
         └─ Same mechanism

BYTEA column
    └─ Large binary (> ~2KB) → TOAST
         └─ Keeps rows compact
              └─ Fetch only when needed
```

## 4. Kesimpulan

Binary data di PostgreSQL disimpan menggunakan tipe BYTEA, yang menyimpan raw bytes tanpa encoding atau collation. Untuk hash identifiers (seperti MD5), UUID type adalah pilihan terbaik karena memberikan storage paling kecil (16 bytes) tanpa varlena overhead.

**Key Points:**

1. **BYTEA**: Variable-length binary, max 1GB (protocol limit 2GB untuk SELECT)
2. **Output Formats**: Hex (recommended) vs Escape (legacy)
3. **TOAST**: Large binary data otomatis di-TOAST
4. **File Storage**: Large files → S3/R2 + pointer, NOT in database
5. **Hashes**: MD5 (not secure!) vs SHA-256 (secure)
6. **Storage Sizes**: UUID (16B) < BYTEA (20B) < TEXT (36B)

**Best Practices:**

- ✅ Gunakan UUID untuk MD5/128-bit hashes (smallest & fastest!)
- ✅ Gunakan BYTEA untuk flexibility (SHA-256, arbitrary binary)
- ✅ Store large files di file storage (S3, R2, Tigris)
- ✅ Simpan metadata dan pointers di database
- ✅ Set `bytea_output = 'hex'` untuk consistency
- ❌ Jangan store files > 1MB di database
- ❌ Jangan gunakan MD5 untuk security (use SHA-256+)
- ❌ Jangan gunakan TEXT untuk hashes (wasteful!)

**Storage Efficiency untuk MD5 Hashes (VERIFIED!):**

```
UUID (MD5 cast):          16 bytes  ✅ SMALLEST & BEST!
    - Fixed-size type
    - No varlena overhead
    - Native PostgreSQL optimization
    - Best index performance

BYTEA (decoded):          20 bytes  ✅ Flexible
    - Variable-length type
    - 4 bytes varlena overhead
    - Works with any hash size

TEXT (hex string):        36 bytes  ❌ WASTEFUL
    - Variable-length type
    - 4 bytes varlena overhead
    - 32 bytes for hex representation
    - Only use when human-readability is critical
```
