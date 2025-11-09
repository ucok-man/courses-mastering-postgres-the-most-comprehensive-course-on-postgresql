# 📝 Binary Data di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas tentang penyimpanan binary data (data mentah dalam bentuk bytes) di PostgreSQL menggunakan tipe data `BYTEA`. Berbeda dengan TEXT yang menyimpan string dengan encoding dan collation, BYTEA menyimpan raw bytes tanpa interpretasi karakter. Video menjelaskan kapan sebaiknya menggunakan binary data, use cases yang valid (seperti checksums dan hashes), dan perbandingan ukuran storage antara berbagai representasi hash (MD5 sebagai text, BYTEA, dan UUID).

## 2. Konsep Utama

### a. Tipe Data BYTEA

BYTEA (byte array) adalah tipe data PostgreSQL untuk menyimpan binary data mentah.

**Karakteristik BYTEA:**

- Variable-length column (ukuran berubah sesuai data)
- Maksimum: 1 GB per value
- Tidak ada encoding atau collation (raw bytes)
- TOAST-enabled untuk data besar
- Comparison berdasarkan byte sequence, bukan karakter

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

| Aspek          | TEXT               | BYTEA            |
| -------------- | ------------------ | ---------------- |
| **Content**    | Characters         | Raw bytes        |
| **Encoding**   | UTF-8, Latin1, dll | No encoding      |
| **Collation**  | en_US.UTF-8, dll   | No collation     |
| **Comparison** | Character rules    | Byte-by-byte     |
| **Use case**   | Human-readable     | Machine data     |
| **Overhead**   | Encoding info      | Minimal overhead |

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
-- 1. Database bloat
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

-- Perbandingan ukuran
SELECT
    pg_column_size(MD5('Hello World')) AS md5_text_size,
    pg_column_size(decode(MD5('Hello World'), 'hex')) AS md5_bytea_size;
-- Output:
-- md5_text_size: 36 bytes (TEXT overhead)
-- md5_bytea_size: 20 bytes (BYTEA overhead)
```

**Why Convert?**

1. **Ukuran lebih kecil**: 20 bytes vs 36 bytes
2. **Comparison lebih cepat**: Byte comparison vs character comparison
3. **Tidak ada collation overhead**: Raw bytes
4. **Better for indexing**: Fixed-size binary

**Process:**

```
'Hello World'
    ↓ MD5()
'3e25960a79dbc69b674cd4ec67a72c62' (32 hex chars as TEXT)
    ↓ decode(..., 'hex')
\x3e25960a79dbc69b674cd4ec67a72c62 (16 bytes as BYTEA)
```

### h. UUID: Best of Both Worlds

PostgreSQL punya tipe UUID yang optimal untuk menyimpan hash.

```sql
-- Cast MD5 ke UUID
SELECT MD5('Hello World')::UUID;
-- Output: 3e25960a-79db-c69b-674c-d4ec67a72c62

-- Format UUID dengan dashes, tapi storage efficient!

-- Perbandingan ukuran storage
SELECT
    pg_column_size(MD5('Hello World')) AS text_size,           -- 36 bytes
    pg_column_size(decode(MD5('Hello World'), 'hex')) AS bytea_size,  -- 20 bytes
    pg_column_size(MD5('Hello World')::UUID) AS uuid_size;    -- 16 bytes ✅
-- uuid_size: 16 bytes (PALING KECIL!)
```

**UUID Format:**

```
MD5 output:     3e25960a79dbc69b674cd4ec67a72c62
UUID format:    3e25960a-79db-c69b-674c-d4ec67a72c62
                └──┬──┘ └─┬┘ └─┬┘ └─┬┘ └────┬────┘
                   │     │    │    │       │
              8 chars 4   4    4    12 chars

Storage: Fixed 16 bytes (paling efficient!)
Display: Formatted dengan dashes (readable!)
```

**Why UUID is Better:**

```sql
CREATE TABLE file_checksums (
    file_id INTEGER,
    checksum_text TEXT,           -- 36 bytes
    checksum_bytea BYTEA,         -- 20 bytes
    checksum_uuid UUID            -- 16 bytes ✅ BEST!
);

-- UUID advantages:
-- 1. Smallest storage (16 bytes)
-- 2. Fast comparison (fixed-size binary)
-- 3. Readable display (formatted dengan dashes)
-- 4. Native PostgreSQL type (better support)
-- 5. Can generate with gen_random_uuid()
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

### Storage Efficiency Hierarchy:

```
Most Efficient
    ↑
UUID (MD5 cast)          16 bytes  ✅ BEST for hashes
    ↑
BYTEA (decoded hex)      20 bytes  ✅ Good
    ↑
TEXT (hex string)        36 bytes  ⚠️ Wasteful
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
├─ Hash / Checksum
│   ├─ Security needed: SHA-256 (BYTEA)
│   └─ Speed needed: MD5 → UUID ✅
│
└─ Files
    ├─ Very small (< 100KB): Maybe BYTEA
    └─ Anything larger: S3/R2 + URL pointer ✅
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

### Evolution dari Previous Videos:

```
Video: Character Types
    TEXT dengan encoding & collation
    ↓
Video: Binary Data
    BYTEA tanpa encoding & collation

Video: Domain Types
    Custom validation untuk TEXT
    ↓
Future: Generated Columns
    Auto-compute hash/checksum
```

## 4. Catatan Tambahan / Insight

### Tips Praktis:

1. **Default ke Hex Output:**

   ```sql
   -- Pastikan hex output di connection string atau init
   SET bytea_output = 'hex';

   -- Atau di postgresql.conf
   bytea_output = 'hex'
   ```

2. **Working dengan Binary Data di Aplikasi:**

   ```python
   # Python example dengan psycopg2
   import psycopg2

   conn = psycopg2.connect(...)
   cur = conn.cursor()

   # Insert binary data
   with open('image.jpg', 'rb') as f:
       binary_data = f.read()
       cur.execute(
           "INSERT INTO files (name, data) VALUES (%s, %s)",
           ('image.jpg', psycopg2.Binary(binary_data))
       )

   # Retrieve binary data
   cur.execute("SELECT data FROM files WHERE name = %s", ('image.jpg',))
   binary_data = bytes(cur.fetchone()[0])
   ```

3. **Checksumming Pattern:**

   ```sql
   -- Store hash alongside content untuk integrity
   CREATE TABLE documents (
       id SERIAL PRIMARY KEY,
       content TEXT,
       content_md5 UUID GENERATED ALWAYS AS (MD5(content)::UUID) STORED
   );

   -- Verify integrity
   SELECT
       id,
       content_md5 = MD5(content)::UUID AS is_valid
   FROM documents;
   -- Jika is_valid = false → corruption detected!
   ```

4. **File Metadata Pattern:**

   ```sql
   -- Best practice: Store metadata di DB, file di S3
   CREATE TABLE uploads (
       id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
       user_id INTEGER NOT NULL,
       original_filename TEXT NOT NULL,
       s3_key TEXT NOT NULL,           -- path di S3
       s3_bucket TEXT NOT NULL,
       file_size BIGINT NOT NULL,
       mime_type TEXT NOT NULL,
       md5_checksum UUID NOT NULL,     -- integrity check
       uploaded_at TIMESTAMP DEFAULT NOW(),

       UNIQUE(md5_checksum)  -- Prevent duplicates
   );

   -- Upload flow:
   -- 1. Calculate MD5 locally
   -- 2. Check if md5_checksum exists
   -- 3. If not, upload to S3
   -- 4. Store metadata di DB
   ```

5. **Binary Data Comparison:**

   ```sql
   -- Exact match (efficient)
   SELECT * FROM files WHERE data = '\x48656c6c6f';

   -- Partial match (TIDAK efficient!)
   SELECT * FROM files WHERE data LIKE '%\x48656c%';
   -- LIKE tidak work well dengan BYTEA!

   -- Better: Use hash untuk comparison
   SELECT * FROM files WHERE content_hash = MD5(...)::UUID;
   ```

### Kesalahan Umum:

```sql
-- ❌ KESALAHAN 1: Store large images di BYTEA
CREATE TABLE products (
    image BYTEA  -- 2MB per product
);
-- 10,000 products = 20GB di database!

-- ✅ FIX:
CREATE TABLE products (
    image_url TEXT  -- Link ke CDN
);


-- ❌ KESALAHAN 2: Gunakan MD5 untuk passwords
CREATE TABLE users (
    password_hash BYTEA  -- MD5(password) ❌ BROKEN!
);

-- ✅ FIX: Gunakan bcrypt atau argon2
CREATE EXTENSION pgcrypto;
CREATE TABLE users (
    password_hash TEXT  -- crypt(password, gen_salt('bf'))
);


-- ❌ KESALAHAN 3: Compare binary data tanpa index
SELECT * FROM files WHERE data = some_large_binary;
-- Sequential scan yang sangat lambat!

-- ✅ FIX: Index pada hash
CREATE INDEX idx_files_hash ON files(content_hash);
SELECT * FROM files WHERE content_hash = MD5(...)::UUID;


-- ❌ KESALAHAN 4: Lupa BYTEA overhead
-- MD5 = 16 bytes, tapi storage bukan 16 bytes
SELECT pg_column_size(decode(MD5('test'), 'hex'));
-- Output: 20 bytes (ada 4 bytes overhead)

-- ✅ SOLUTION: Gunakan UUID untuk true 16 bytes
SELECT pg_column_size(MD5('test')::UUID);
-- Output: 16 bytes ✅


-- ❌ KESALAHAN 5: Insert text langsung ke BYTEA
INSERT INTO files VALUES ('\x48656c6c6f');  -- OK
INSERT INTO files VALUES ('Hello');          -- ❌ ERROR!
-- ERROR: column "data" is of type bytea but expression is of type text

-- ✅ FIX: Explicit conversion
INSERT INTO files VALUES (convert_to('Hello', 'UTF8'));
-- Atau
INSERT INTO files VALUES ('\x48656c6c6f'::BYTEA);
```

### Performance Considerations:

```sql
-- Benchmark: Hash comparison methods
-- Dataset: 1 million rows

-- Method 1: TEXT comparison
CREATE TABLE hashes_text (hash TEXT);
CREATE INDEX ON hashes_text(hash);
-- Lookup: ~5ms (collation overhead)

-- Method 2: BYTEA comparison
CREATE TABLE hashes_bytea (hash BYTEA);
CREATE INDEX ON hashes_bytea(hash);
-- Lookup: ~3ms (raw byte comparison)

-- Method 3: UUID comparison
CREATE TABLE hashes_uuid (hash UUID);
CREATE INDEX ON hashes_uuid(hash);
-- Lookup: ~2ms ✅ (fixed-size, optimized)

-- Winner: UUID untuk hash storage!
```

### Advanced Patterns:

**Pattern 1: Content-Addressable Storage**

```sql
-- Store content once, reference many times
CREATE TABLE blobs (
    hash UUID PRIMARY KEY,
    content BYTEA NOT NULL
);

CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content_hash UUID REFERENCES blobs(hash)
);

-- Same content = same hash = stored once only ✅
```

**Pattern 2: Hybrid Storage**

```sql
-- Small files in DB, large in S3
CREATE TABLE files (
    id UUID PRIMARY KEY,
    name TEXT,
    size INTEGER,

    -- Small files (< 100KB)
    inline_data BYTEA CHECK (
        (size <= 102400 AND inline_data IS NOT NULL) OR
        (size > 102400 AND inline_data IS NULL)
    ),

    -- Large files (> 100KB)
    s3_key TEXT CHECK (
        (size > 102400 AND s3_key IS NOT NULL) OR
        (size <= 102400 AND s3_key IS NULL)
    )
);

-- Application logic:
-- If size <= 100KB → store in inline_data
-- If size > 100KB → upload to S3, store s3_key
```

**Pattern 3: Integrity Verification**

```sql
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    s3_key TEXT,
    expected_md5 UUID,
    last_verified TIMESTAMP
);

-- Periodic integrity check
CREATE FUNCTION verify_file_integrity(file_id INTEGER)
RETURNS BOOLEAN AS $$
DECLARE
    actual_md5 UUID;
BEGIN
    -- Fetch dari S3, calculate MD5
    actual_md5 := ... -- S3 operation

    UPDATE files
    SET last_verified = NOW()
    WHERE id = file_id
      AND expected_md5 = actual_md5;

    RETURN FOUND;
END;
$$ LANGUAGE plpgsql;
```

### Analogi:

**BYTEA vs TEXT seperti:**

- **TEXT** = Dokumen Word (formatted, ada metadata, human-readable)
- **BYTEA** = File ZIP (compressed binary, machine-readable)

**Storage Decision seperti:**

- **Database** = Brankas bank (secure, expensive, untuk valuables)
- **File Storage** = Gudang (cheap, spacious, untuk bulk items)
- **Small files di DB** = Perhiasan di brankas ✅
- **Large files di DB** = Furniture di brankas ❌

**Hash Types seperti:**

- **MD5** = Kunci rumah lama (fast tapi tidak secure)
- **SHA-256** = Kunci modern dengan chip (slower tapi secure)
- **UUID** = Kunci dengan barcode (readable dan efficient)

## 5. Kesimpulan

Binary data di PostgreSQL disimpan menggunakan tipe BYTEA, yang menyimpan raw bytes tanpa encoding atau collation. Ini sangat berguna untuk checksums, hashes, dan small binary data, tapi TIDAK recommended untuk large files.

**Key Points:**

1. **BYTEA**: Variable-length binary column, max 1GB (tapi jangan)
2. **Output Formats**: Hex (recommended) vs Escape (legacy)
3. **TOAST**: Large binary data otomatis di-TOAST
4. **File Storage**: Large files → S3/R2 + pointer, NOT in database
5. **Hashes**: MD5 (not secure!) vs SHA-256 (secure), store as UUID untuk efficiency

**Best Practices:**

- ✅ Gunakan BYTEA untuk checksums dan hashes
- ✅ Store large files di file storage (S3, R2, Tigris)
- ✅ Simpan metadata dan pointers di database
- ✅ Gunakan UUID type untuk menyimpan MD5/hash (most efficient)
- ✅ Set `bytea_output = 'hex'` untuk consistency
- ❌ Jangan store files > 1MB di database
- ❌ Jangan gunakan MD5 untuk security (use SHA-256+)

**Storage Efficiency untuk Hashes:**

```
TEXT (MD5 string):    36 bytes  ❌
BYTEA (decoded):      20 bytes  ⚠️
UUID (MD5 cast):      16 bytes  ✅ BEST!
```

**Decision Guide:**

- Need security → SHA-256 (BYTEA)
- Need speed & deduplication → MD5 → UUID
- Need to store files → S3 + pointer (TEXT)
- Need checksums → MD5/SHA-256 → UUID

**Key takeaway:** BYTEA adalah tool yang powerful untuk binary data, tapi harus digunakan dengan bijak. Untuk large files, selalu gunakan dedicated file storage. Untuk hashes dan checksums, UUID type memberikan best balance antara storage efficiency dan usability. Remember: MD5 is fast but NOT secure - use only for non-cryptographic purposes!
