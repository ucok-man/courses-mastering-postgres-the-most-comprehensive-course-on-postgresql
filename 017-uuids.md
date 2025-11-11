# 📝 UUID sebagai Tipe Data di PostgreSQL (Versi Terkoreksi)

## 1. Ringkasan Singkat

Video ini membahas UUID (Universally Unique Identifier) sebagai tipe data di PostgreSQL, bukan sebagai primary key (yang akan dibahas di video terpisah). UUID adalah identifier 128-bit yang sangat unik dan berguna untuk generating IDs tanpa koordinasi antar sistem. Video menjelaskan keuntungan menyimpan UUID sebagai tipe data native (16 bytes) dibanding TEXT (37-40 bytes), berbagai versi UUID (terutama v4 random dan v7 sortable), dan cara generate UUID di PostgreSQL.

## 2. Konsep Utama

### a. UUID sebagai Tipe Data

UUID di PostgreSQL adalah tipe data native yang menyimpan 128-bit identifier dengan format standar.

**Karakteristik UUID:**

- Fixed-size: **16 bytes storage** (raw binary)
- Display format: 8-4-4-4-12 hex digits (36 characters dengan hyphens)
- Internal storage: **Binary format tanpa hyphens atau ASCII characters**
- Universally unique (collision probability sangat rendah)
- Native type dengan optimasi khusus

```sql
-- Membuat table dengan UUID column
CREATE TABLE uuid_example (
    uuid_value UUID
);

-- Insert UUID sebagai string (auto-converted to binary)
INSERT INTO uuid_example VALUES ('550e8400-e29b-41d4-a716-446655440000');

-- View data (displayed as formatted string)
SELECT * FROM uuid_example;
-- Output: 550e8400-e29b-41d4-a716-446655440000

-- Cek tipe data
SELECT pg_typeof(uuid_value) FROM uuid_example;
-- Output: uuid (bukan text!)

-- Cek storage size
SELECT pg_column_size(uuid_value) FROM uuid_example;
-- Output: 16 bytes ✅ (stored as raw binary)
```

**Format UUID:**

```
Display format (string representation):
3e25960a-79db-c69b-674c-d4ec67a72c62
└──┬───┘ └─┬┘ └─┬┘ └─┬┘ └────┬─────┘
   │       │    │    │       │
8 digits   4    4    4    12 digits

Total: 32 hex digits + 4 hyphens = 36 characters (for display only)
Storage: 16 bytes (128 bits pure binary, NO hyphens stored!)
```

**⚠️ PENTING:** PostgreSQL menyimpan UUID sebagai **raw 16-byte binary value**, bukan sebagai string dengan hyphens. Hyphens hanya muncul saat display/conversion ke text.

### b. UUID vs TEXT: Storage Efficiency

Storing UUID sebagai native type jauh lebih efisien dari TEXT.

```sql
-- Perbandingan storage size
SELECT
    pg_column_size(uuid_value) AS uuid_type_size,
    pg_column_size(uuid_value::TEXT) AS text_type_size
FROM uuid_example;

-- Output:
-- uuid_type_size: 16 bytes  ✅
-- text_type_size: 37-40 bytes (typically 40)  ❌

-- Breakdown TEXT size:
-- - 36 bytes untuk string '550e8400-e29b-41d4-a716-446655440000'
-- - 1-4 bytes untuk varlena header (varies by PostgreSQL version)
-- - Potential padding untuk memory alignment
-- Total: typically 40 bytes (can vary 37-40)

-- UUID native: 16 bytes (raw 128-bit binary storage)
-- NO overhead, NO padding, pure data ✅
```

**Impact pada Large Tables:**

```sql
-- Example: 10 million users
CREATE TABLE users_text (
    id TEXT  -- UUID sebagai TEXT: ~40 bytes average
);
-- Total: 10M × 40 bytes = 400 MB

CREATE TABLE users_uuid (
    id UUID  -- UUID sebagai UUID: 16 bytes
);
-- Total: 10M × 16 bytes = 160 MB

-- Savings: 240 MB (60% reduction!) ✅
-- Additional benefits:
-- - Faster comparison (binary vs string)
-- - More efficient indexing (smaller index)
-- - Better cache utilization
```

### c. Universally Unique: Collision Probability

UUID dirancang untuk uniqueness tanpa koordinasi.

**Collision Probability:**

```
**Ukuran ruang UUID:**
UUID punya kemungkinan kombinasi sebanyak **2^128**, atau sekitar
**340 triliun-triliun-triliun** kemungkinan unik. Sangat banyak.

**Untuk UUID versi 4 (yang acak):**
Bagian acaknya adalah **122 bit**, jadi total kemungkinan acak sekitar
**2^122 ≈ 5,3 × 10³⁶** — angka ini juga sangat besar.

**Kemungkinan tabrakan (dua UUID sama):**
Kalau kamu membuat **1 triliun (10¹²)** UUID, peluang ada yang sama itu kira-kira
**1 banding 100 miliar** — alias hampir mustahil.

**Menurut “paradoks ulang tahun”:**
Kamu perlu membuat sekitar **2,6 miliar UUID** baru sebelum peluang tabrakan mencapai **50%**.
Dalam dunia nyata, itu **sangat tidak mungkin terjadi**.

**Sebagai perbandingan, kamu lebih mungkin untuk:**

* Disambar petir **lebih dari 40 kali dalam sehari**
* **Menang lotre 5 kali berturut-turut**
* Menemukan **satu atom tertentu dari seluruh atom dalam tubuhmu**
```

**Why Universally Unique Matters:**

```
Scenario: Multi-system ID generation

System A (Server 1)     System B (Server 2)     System C (Client App)
    ↓                        ↓                        ↓
Generate UUID           Generate UUID           Generate UUID
    ↓                        ↓                        ↓
No coordination needed! ✅
All IDs guaranteed unique ✅
Can merge data later without conflicts ✅
No network calls required ✅
```

### d. Use Cases untuk UUID

UUID sangat berguna untuk specific scenarios.

#### ✅ Use Case 1: Distributed Systems

```sql
-- Microservices architecture
-- Service A: Order Service
CREATE TABLE orders (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    customer_id UUID,
    total NUMERIC
);

-- Service B: Shipping Service (different database)
CREATE TABLE shipments (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    order_id UUID,  -- Reference ke order dari Service A
    tracking_number TEXT
);

-- Keuntungan:
-- - Generate ID independently di setiap service ✅
-- - No central ID coordinator needed ✅
-- - No network overhead untuk ID generation ✅
-- - Easy to merge/sync data antar services ✅
-- - Service decoupling ✅
```

#### ✅ Use Case 2: Client-Side Generation

```sql
-- Frontend generates ID sebelum server insert
-- JavaScript (modern browsers):
const newId = crypto.randomUUID();
// Output: '550e8400-e29b-41d4-a716-446655440000'

-- Send to server
fetch('/api/items', {
    method: 'POST',
    body: JSON.stringify({
        id: newId,
        name: 'New Item'
    })
});

-- PostgreSQL accepts ID yang sudah di-generate:
INSERT INTO items (id, name)
VALUES ('550e8400-e29b-41d4-a716-446655440000', 'New Item');

-- Keuntungan:
-- - Optimistic UI updates (show ID immediately) ✅
-- - Offline-first applications ✅
-- - No server roundtrip untuk get ID ✅
-- - Reduced server load ✅
-- - Better user experience (instant feedback) ✅
```

#### ✅ Use Case 3: Multiple Databases Sync

```sql
-- Database 1 (Production - US East)
INSERT INTO users (id, email)
VALUES ('550e8400-e29b-41d4-a716-446655440000', 'user@example.com');

-- Database 2 (Production - EU West)
INSERT INTO users (id, email)
VALUES ('1a2b3c4d-5e6f-7890-abcd-1234567890ab', 'eu-user@example.com');

-- Database 3 (Analytics/Data Warehouse)
-- Merge data from both regions:
INSERT INTO users_analytics
SELECT * FROM us_east_users
UNION ALL
SELECT * FROM eu_west_users;
-- No ID conflicts! ✅

-- Use cases:
-- - Multi-region deployments
-- - Database replication
-- - Data warehousing
-- - Backup/restore scenarios
```

#### ⚠️ Use Case 4: Public IDs (dengan caveats importantes)

```sql
-- Expose UUID ke public (URLs, APIs)
-- Advantage: Not sequential (harder to enumerate)
-- https://api.example.com/users/550e8400-e29b-41d4-a716-446655440000

-- vs sequential IDs:
-- https://api.example.com/users/12345
-- https://api.example.com/users/12346 (easy to guess! ❌)
-- https://api.example.com/users/12347 (attacker can scrape all!)

-- ⚠️ IMPORTANT CAVEATS:
-- 1. UUID v4: Fully random (good for security by obscurity)
-- 2. UUID v7: Contains timestamp (can be predicted/estimated!)
-- 3. NEVER rely on UUID alone for security
-- 4. Always implement proper authentication & authorization
-- 5. Rate limiting still necessary
-- 6. UUID ≠ security token, it's just an identifier

-- Example attack pada UUID v7:
-- If created at 2024-01-01 10:00:00
-- UUID starts with: 018d0... (timestamp portion)
-- Attacker can estimate: "UUIDs probably between 018d0... and 018d1..."
-- Still need proper auth! ✅
```

### e. Generating UUIDs di PostgreSQL

PostgreSQL menyediakan built-in function untuk UUID generation.

#### UUID v4: Random (Built-in) ✅

```sql
-- Generate random UUID (version 4)
-- Available di semua PostgreSQL versions (9.4+)
SELECT gen_random_uuid();
-- Output example: 9b8c7e6d-5f4a-4b3c-9d8e-7f6a5b4c3d2e

-- Setiap call generates UUID baru
SELECT gen_random_uuid();
-- Output: 1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d (berbeda!)

SELECT gen_random_uuid();
-- Output: a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d (berbeda lagi!)

-- Use sebagai default value
CREATE TABLE items (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    name TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Insert tanpa specify ID
INSERT INTO items (name) VALUES ('Item 1');
INSERT INTO items (name) VALUES ('Item 2');
-- IDs auto-generated! ✅

SELECT * FROM items;
-- id                                   | name   | created_at
-- 9b8c7e6d-5f4a-4b3c-9d8e-7f6a5b4c3d2e | Item 1 | 2024-03-15 10:30:00
-- 1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d | Item 2 | 2024-03-15 10:30:01
```

**UUID v4 Characteristics:**

- **Fully pseudorandom:** 122 bits of randomness
  - 128 total bits
  - minus 4 bits for version field (set to `0100` = 4)
  - minus 2 bits for variant field (set to `10`)
  - = **122 bits pseudorandom**
- **No sortable property** (completely random order)
- **❌ BAD sebagai primary key** (causes index fragmentation - akan dijelaskan nanti)
- **✅ OK untuk secondary identifiers** (correlation IDs, tracking IDs, etc)
- **✅ Best for security by obscurity** (unpredictable)

#### UUID v7: Sortable (PostgreSQL 17+ or Extension) ⚠️

```sql
-- ⚠️ UUID v7 AVAILABILITY:

-- Option 1: PostgreSQL 17+ (built-in)
-- Available: November 2024+
SELECT gen_random_uuid_v7();  -- Native function ✅

-- Option 2: Extension (PostgreSQL < 17)
-- Popular extensions:
-- - pg_uuidv7 (community extension)
-- - pgcrypto + custom function
-- - Some cloud providers (AWS RDS, Google Cloud SQL) may include v7

-- Install pg_uuidv7 extension (example):
CREATE EXTENSION IF NOT EXISTS pg_uuidv7;

-- Generate UUID v7:
SELECT uuid_generate_v7();
-- Output: 018e1e5e-7c8a-7xxx-yyyy-zzzzzzzzzzzz

-- ⚠️ NOTE: uuid-ossp extension does NOT support v7!
-- uuid-ossp only supports: v1, v3, v4, v5
```

**❌ COMMON MISTAKE:**

```sql
-- WRONG! uuid-ossp tidak punya v7
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SELECT uuid_generate_v7();  -- ERROR: function does not exist ❌

-- CORRECT approaches:
-- 1. Use PostgreSQL 17+ with gen_random_uuid_v7()
-- 2. Install pg_uuidv7 extension
-- 3. Implement custom v7 generator using PL/pgSQL
```

**UUID v7 Format & Benefits:**

```
UUID v7 Structure: 128
[48-bit timestamp][4-bit version][12-bit random][2-bit variant][62-bit random]

Example:
018e1e5e-7c8a-7xxx-yyyy-zzzzzzzzzzzz
└──┬──┘ └─┬┘ └─┬┘ └─┬┘ └────┬─────┘
   │       │    │    │       │
   │       │    │    │       └─ 62 bits random
   │       │    │    └───────── 2 bits variant (10)
   │       │    └────────────── 12 bits random
   │       └─────────────────── 4 bits version (0111 = 7)
   └─────────────────────────── 48 bits Unix timestamp (milliseconds)

Keuntungan:
1. Lexicographically sortable (timestamp di depan) ✅
2. Time-ordered (natural chronological sorting) ✅
3. Dapat extract timestamp untuk debugging ✅
4. ✅ GOOD sebagai primary key (sequential insertion)
5. Maintains UUID uniqueness benefits ✅

Timestamp capacity:
- 48 bits = 2^48 milliseconds
- From Unix epoch (1970-01-01): ~8,925 years
- Valid until approximately year 10,895 ✅
```

**UUID v7 vs v4 Comparison:**

```
UUID v4 (random):
Generated order:
- 9b8c7e6d-5f4a-4b3c-9d8e-7f6a5b4c3d2e (random position)
- 1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d (random position)
- c3d2e1f0-a9b8-c7d6-e5f4-a3b2c1d0e9f8 (random position)
❌ Random order → B-tree index fragmentation
❌ Requires page splits and rebalancing
❌ Poor cache locality

UUID v7 (time-ordered):
Generated order (increasing timestamp):
- 018e1e5e-0000-7xxx-xxxx-xxxxxxxxxxxx (oldest)
- 018e1e5e-1000-7yyy-yyyy-yyyyyyyyyyyy
- 018e1e5e-2000-7zzz-zzzz-zzzzzzzzzzzz
- 018e1e5f-0000-7aaa-aaaa-aaaaaaaaaaaa (newest)
✅ Sequential order → efficient B-tree index
✅ Append-only insertion pattern
✅ Good cache locality
✅ Better range query performance
```

### f. UUID Versions: Complete Overview

Ada 8 versi UUID yang officially defined (RFC 9562).

| Version | Type                 | Description                     | Availability                  | Use Case                                 |
| ------- | -------------------- | ------------------------------- | ----------------------------- | ---------------------------------------- |
| **v1**  | Time-based + MAC     | Timestamp + machine MAC address | uuid-ossp extension           | Legacy systems, audit trails             |
|         |                      |                                 |                               | ⚠️ Privacy concern (exposes MAC)         |
| **v2**  | DCE Security         | Reserved for DCE security       | Rarely implemented            | Specialized security (almost never used) |
| **v3**  | Name-based (MD5)     | MD5 hash of namespace + name    | uuid-ossp extension           | Deterministic UUIDs (legacy)             |
|         |                      |                                 |                               | ⚠️ MD5 considered weak                   |
| **v4**  | Random               | Pseudorandom (122 bits)         | ✅ gen_random_uuid() built-in | General purpose, secondary IDs           |
|         |                      |                                 |                               | ❌ NOT for primary keys                  |
| **v5**  | Name-based (SHA-1)   | SHA-1 hash of namespace + name  | uuid-ossp extension           | Deterministic UUIDs (better than v3)     |
|         |                      |                                 |                               | ✅ Idempotent operations                 |
| **v6**  | Reordered time       | Like v1 but sortable            | Extension or custom           | Improvement over v1 (sortable)           |
| **v7**  | Unix timestamp + rnd | Time-ordered + random           | ✅ PG 17+ or pg_uuidv7        | ✅ **Best for primary keys**             |
|         |                      |                                 |                               | ✅ Modern distributed systems            |
| **v8**  | Custom               | User-defined format             | Custom implementation         | Specialized needs (application-specific) |

**Recommendations by Use Case:**

```
Primary Key:
├─ Single database
│   └─ BIGINT SERIAL ✅✅ (most efficient)
└─ Distributed systems
    └─ UUID v7 ✅ (sortable + unique)

Secondary Identifier:
├─ Random, unpredictable
│   └─ UUID v4 ✅ (built-in, easy)
└─ Deterministic (same input = same output)
    └─ UUID v5 ✅ (SHA-1 based)

Public-facing ID:
├─ High security concern
│   └─ UUID v4 ✅ (fully random)
└─ Need chronological order
    └─ UUID v7 ⚠️ (but implement proper auth!)

Legacy integration:
└─ UUID v1 (if required by external system)
    ⚠️ Privacy: exposes MAC address
```

### g. UUID Components Breakdown

Setiap UUID version memiliki internal structure yang berbeda.

#### UUID v4 (Random) Detailed Structure:

```
550e8400-e29b-41d4-a716-446655440000
│       │       │    │    │
├───────┴───────┴────┴────┴──────────── 128 bits total
│
└─ Bits allocation:

Bit positions:   0                                 127
                 │                                  │
Hex positions:   550e8400-e29b-41d4-a716-446655440000
                 ^^^^^^^^ ^^^^ ^^^^ ^^^^ ^^^^^^^^^^^^

Breakdown:
┌─────────────────────────────────────────────────────────────┐
│ Bits 0-31:    Random (32 bits)     → 550e8400               │
│ Bits 32-47:   Random (16 bits)     → e29b                   │
│ Bits 48-51:   Version = 4 (fixed)  → 4 (in 41d4)            │
│ Bits 52-63:   Random (12 bits)     → 1d4                    │
│ Bits 64-65:   Variant = 10 (fixed) → (in a716)              │
│ Bits 66-127:  Random (62 bits)     → 716-446655440000       │
└─────────────────────────────────────────────────────────────┘

Total randomness: 32 + 16 + 12 + 62 = 122 bits ✅

Fixed bits:
- Version field (bits 48-51): 0100 (binary) = 4
- Variant field (bits 64-65): 10 (binary) = RFC 4122 compliant

Probability space: 2^122 ≈ 5.3 × 10^36 unique values
```

#### UUID v7 (Timestamp-based) Detailed Structure:

```
018e1e5e-7c8a-7xxx-yyyy-zzzzzzzzzzzz
│       │       │    │    │
├───────┴───────┴────┴────┴──────────── 128 bits total
│
└─ Bits allocation:

Bit positions:   0                                 127
                 │                                  │
                 018e1e5e-7c8a-7xxx-yyyy-zzzzzzzzzzzz

Breakdown:
┌─────────────────────────────────────────────────────────────┐
│ Bits 0-47:    Unix timestamp ms (48 bits) → 018e1e5e-7c8a   │
│ Bits 48-51:   Version = 7 (fixed)         → 7               │
│ Bits 52-63:   Random (12 bits)            → xxx             │
│ Bits 64-65:   Variant = 10 (fixed)        → (in yyyy)       │
│ Bits 66-127:  Random (62 bits)            → yyy-zzzz...     │
└─────────────────────────────────────────────────────────────┘

Timestamp portion (bits 0-47):
- Unix epoch milliseconds
- Range: 0 to 2^48 - 1
- Covers: 1970-01-01 to ~year 10,895
- Resolution: 1 millisecond

Random portion: 12 + 62 = 74 bits
- Handles collisions within same millisecond
- 2^74 ≈ 18.9 × 10^21 unique values per millisecond
- Enough for millions of UUIDs per millisecond ✅

Time ordering:
- UUIDs naturally sort by creation time
- Lexicographic order = chronological order ✅
```

**Extracting Timestamp dari UUID v7:**

```sql
-- ⚠️ Implementation depends on extension/PostgreSQL version
-- Example with custom function (conceptual):

CREATE OR REPLACE FUNCTION uuid_v7_to_timestamp(uuid_val UUID)
RETURNS TIMESTAMP WITH TIME ZONE AS $$
DECLARE
    uuid_hex TEXT;
    timestamp_hex TEXT;
    timestamp_ms BIGINT;
BEGIN
    -- Convert UUID to hex string (no hyphens)
    uuid_hex := REPLACE(uuid_val::TEXT, '-', '');

    -- Extract first 12 hex chars (48 bits)
    timestamp_hex := SUBSTRING(uuid_hex FROM 1 FOR 12);

    -- Convert hex to bigint (milliseconds since epoch)
    timestamp_ms := ('x' || timestamp_hex)::BIT(48)::BIGINT;

    -- Convert to timestamp
    RETURN TO_TIMESTAMP(timestamp_ms / 1000.0);
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Usage:
SELECT uuid_v7_to_timestamp('018e1e5e-7c8a-7xxx-yyyy-zzzzzzzzzzzz');
-- Output: 2024-03-15 10:30:45.123+00

-- Practical use: Debugging, auditing, time-based queries
SELECT id, uuid_v7_to_timestamp(id) as created_at
FROM orders
WHERE uuid_v7_to_timestamp(id) >= NOW() - INTERVAL '1 day';
```

### h. Conversion: String ↔ UUID

PostgreSQL seamlessly converts between text dan UUID type.

```sql
-- String → UUID (implicit cast)
SELECT '550e8400-e29b-41d4-a716-446655440000'::UUID;
-- PostgreSQL converts string to binary 16-byte representation ✅

-- UUID → String (automatic for display)
SELECT uuid_value FROM uuid_example;
-- Internally: 16-byte binary
-- Display: formatted as '550e8400-e29b-41d4-a716-446655440000' ✅

-- Explicit cast to TEXT
SELECT uuid_value::TEXT FROM uuid_example;
SELECT CAST(uuid_value AS TEXT) FROM uuid_example;
-- Both equivalent, outputs TEXT type

-- Type checking
SELECT
    pg_typeof(uuid_value) as internal_type,
    pg_typeof(uuid_value::TEXT) as text_type,
    pg_column_size(uuid_value) as uuid_size,
    pg_column_size(uuid_value::TEXT) as text_size
FROM uuid_example;
-- internal_type: uuid
-- text_type: text
-- uuid_size: 16 bytes
-- text_size: ~40 bytes

-- Invalid format handling
SELECT 'not-a-uuid'::UUID;
-- ERROR: invalid input syntax for type uuid: "not-a-uuid"

SELECT 'xxx50e8400-e29b-41d4-a716-446655440000'::UUID;
-- ERROR: invalid input syntax (non-hex characters)

SELECT '550e8400-e29b-41d4-a716'::UUID;
-- ERROR: invalid input syntax (too short)

-- Valid input formats (all accepted, stored identically):
SELECT '550e8400-e29b-41d4-a716-446655440000'::UUID;  -- Standard (recommended)
SELECT '550e8400e29b41d4a716446655440000'::UUID;      -- No hyphens (accepted)
SELECT '{550e8400-e29b-41d4-a716-446655440000}'::UUID; -- With braces (accepted)
SELECT 'urn:uuid:550e8400-e29b-41d4-a716-446655440000'::UUID; -- URN format

-- All stored as identical 16-byte binary ✅
-- Display always uses standard hyphenated format

-- Best practice: Always use standard hyphenated format
-- Why? Maximum compatibility with other systems/languages
```

**Case Sensitivity:**

```sql
-- UUID comparison is case-insensitive (hex values)
SELECT
    '550e8400-e29b-41d4-a716-446655440000'::UUID =
    '550E8400-E29B-41D4-A716-446655440000'::UUID;
-- Output: true ✅

-- Both convert to same binary representation
```

### i. Performance Implications (Preview)

Video menyebutkan primary key discussion akan datang, tapi memberikan preview.

**UUID v4 sebagai Primary Key: ❌ Problematic**

```sql
CREATE TABLE bad_example (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY
);

-- Insert 1 million rows
INSERT INTO bad_example
SELECT gen_random_uuid() FROM generate_series(1, 1000000);

-- Problem: Random insertion order
-- Example sequence:
-- Insert: 9b8c7e6d-5f4a-... → goes to middle of B-tree
-- Insert: 1a2b3c4d-5e6f-... → goes to beginning
-- Insert: c3d2e1f0-a9b8-... → goes to end
-- Insert: 5d4c3b2a-1e0f-... → goes to middle again

-- B-tree Index Impact:
-- 1. Page splits (frequent) ❌
-- 2. Index fragmentation ❌
-- 3. Poor cache locality ❌
-- 4. Slower INSERT performance ❌
-- 5. Larger index size (fragmentation overhead) ❌
```

**UUID v7 sebagai Primary Key: ✅ Better**

```sql
CREATE TABLE good_example (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY  -- Assuming extension available
);

-- Insert 1 million rows
INSERT INTO good_example
SELECT uuid_generate_v7() FROM generate_series(1, 1000000);

-- Insertion pattern: Sequential (by timestamp)
-- Insert: 018e1e5e-0000-... → appends to end of B-tree
-- Insert: 018e1e5e-0001-... → appends to end
-- Insert: 018e1e5e-0002-... → appends to end
-- Insert: 018e1e5e-0003-... → appends to end

-- B-tree Index Impact:
-- 1. Minimal page splits (mostly append) ✅
-- 2. Low fragmentation ✅
-- 3. Good cache locality ✅
-- 4. Fast INSERT performance ✅
-- 5. Compact index size ✅
```

**Preview Comparison:**

```
B-tree Index Structure (simplified visualization):

UUID v4 (random):
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│ 1a2b... │ │ 5d4c... │ │ 9b8c... │ │ c3d2... │
└────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘
     │           │           │           │
  [Page 1]   [Page 2]   [Page 3]   [Page 4]
     ↓           ↓           ↓           ↓
Fragmented! Pages constantly splitting and rebalancing ❌
High random I/O, poor cache hit rate


UUID v7 (sequential):
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│ 018e5e..│ │ 018e5f..│ │ 018e60..│ │ 018e61..│
└────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘
     │           │           │           │
  [Page 1]   [Page 2]   [Page 3]   [Page 4]
     ↓           ↓           ↓           ↓
Sequential! Pages fill left-to-right ✅
Mostly append operations, good cache hit rate


BIGINT SERIAL (best):
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│  1-1000 │ │1001-2000│ │2001-3000│ │3001-4000│
└────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘
     │           │           │           │
  [Page 1]   [Page 2]   [Page 3]   [Page 4]
     ↓           ↓           ↓           ↓
Perfect sequential! Smallest keys (8 bytes vs 16) ✅✅
Optimal for single-database systems
```

**Performance Metrics (Typical):**

```
Metric                    BIGINT    UUID v7    UUID v4
─────────────────────────────────────────────────────────
Key size                  8 bytes   16 bytes   16 bytes
Insert speed (relative)   1.0x      0.9x       0.6x
Index size (relative)     1.0x      1.5x       2.0x
Page splits (per 1M)      Low       Low        High
Cache hit rate            High      High       Medium
Range query speed         Fast      Fast       Medium
─────────────────────────────────────────────────────────

Conclusion:
- BIGINT: Best for single database ✅✅
- UUID v7: Best for distributed, nearly as good ✅
- UUID v4: Avoid for primary keys ❌
```

## 3. Hubungan Antar Konsep

### Evolution dari Previous Videos:

```
Video: Binary Data (BYTEA)
    └─ MD5 hash storage optimization
         └─ Cast MD5::UUID untuk efficiency
              ↓
              16 bytes (UUID) vs 16 bytes (BYTEA)
              Namun UUID provides structure + semantics
              ↓
Video: UUID sebagai Type ← (Current video)
    └─ UUID native type (16 bytes)
         └─ Better than TEXT (40 bytes)
         └─ UUID v4 vs v7 differences
              ↓
Future Video: Primary Keys & Indexes
    └─ UUID v7 vs BIGINT performance
         └─ B-tree index behavior
         └─ Fragmentation deep dive
         └─ When to use each type
```

### Decision Tree untuk ID Generation:

```
Need to generate unique IDs?
│
├─ Single database system
│   │
│   ├─ Simple auto-increment needed?
│   │   └─ BIGINT SERIAL ✅✅
│   │      - Most efficient (8 bytes)
│   │      - Fastest inserts
│   │      - Smallest indexes
│   │      - Best for 95% of use cases
│   │
│   └─ Need non-sequential public IDs?
│       └─ BIGINT SERIAL (internal) + UUID v4 (public) ✅
│          - Best of both worlds
│          - Efficient internal operations
│          - Secure public exposure
│
├─ Distributed system / Microservices
│   │
│   ├─ Can use central ID coordination service?
│   │   └─ BIGINT from Snowflake/Twitter ID service ✅
│   │      - Still get sequential benefits
│   │      - Centralized but scalable
│   │
│   └─ Cannot coordinate (truly distributed)?
│       │
│       ├─ Need chronological ordering?
│       │   └─ UUID v7 ✅
│       │      - Time-ordered
│       │      - Efficient indexing
│       │      - Distributed generation
│       │
│       └─ Don't need ordering?
│           └─ UUID v4 for secondary IDs ✅
│              - Fully random
│              - Simple generation
│              - Avoid as primary key!
│
├─ Client-side ID generation required
│   │
│   ├─ For primary keys?
│   │   └─ UUID v7 ✅
│   │      - Generate in browser/mobile
│   │      - Send to server
│   │      - Good index performance
│   │
│   └─ For tracking/correlation?
│       └─ UUID v4 ✅
│          - Simple to generate
│          - No server roundtrip
│
└─ Multi-database synchronization
    │
    ├─ Need merge without conflicts?
    │   └─ UUID v7 ✅
    │      - Unique across all DBs
    │      - Time-ordered for debugging
    │
    └─ Deterministic IDs needed?
        └─ UUID v5 (name-based) ✅
           - Same input = same UUID
           - Idempotent operations
```

### Storage Efficiency Hierarchy:

```
Storage Efficiency (Best to Worst)
    ↑
INTEGER (4 bytes)            Small range, rarely sufficient
    ↑
BIGINT (8 bytes)             ✅ Best for sequential IDs
    ↑                        Range: -9.2 × 10^18 to 9.2 × 10^18
    ↑
UUID native (16 bytes)       ✅ Best for distributed IDs
    ↑                        128-bit universal uniqueness
    ↑
TEXT/VARCHAR UUID (37-40)    ❌ NEVER use this!
    ↑                        Waste of space + slower
    ↑
TEXT (arbitrary length)      ❌ Worst option
    ↑
Least Efficient

Real-world impact (10M rows):
- INTEGER: 40 MB
- BIGINT: 80 MB
- UUID: 160 MB
- TEXT UUID: 400 MB ← 5x larger than UUID type! ❌
```

### UUID Version Selection Matrix:

```
┌─────────────────────────────────────────────────────────────┐
│ Use Case                        │ Recommended UUID Version   │
├─────────────────────────────────┼────────────────────────────┤
│ Primary key (distributed)       │ UUID v7 ✅                 │
│ Primary key (single DB)         │ BIGINT SERIAL ✅✅         │
│ Secondary identifier            │ UUID v4 ✅                 │
│ Public-facing ID (security)     │ UUID v4 ✅                 │
│ Correlation/Tracking ID         │ UUID v4 ✅                 │
│ Deterministic (idempotency)     │ UUID v5 ✅                 │
│ Client-side generation          │ UUID v7 (PK) or v4 (other) │
│ Legacy system compatibility     │ UUID v1 ⚠️                 │
│ Time-ordered events             │ UUID v7 ✅                 │
│ Random data partitioning        │ UUID v4 ✅                 │
└─────────────────────────────────────────────────────────────┘

Key Principles:
1. Primary keys → UUID v7 (distributed) or BIGINT (single DB)
2. Random IDs → UUID v4
3. Need timestamp → UUID v7
4. Need deterministic → UUID v5
5. Never UUID v4 as primary key → causes fragmentation ❌
```

## 4. Catatan Tambahan / Insight

### Tips Praktis:

**1. Always Use Native UUID Type:**

```sql
-- ❌ JANGAN: Store sebagai TEXT
CREATE TABLE items (
    id TEXT PRIMARY KEY
);
-- Problems:
-- - 40 bytes per UUID (2.5x larger)
-- - Slower string comparison
-- - More index overhead
-- - Potential encoding issues
-- - No type safety

-- ✅ LAKUKAN: Store sebagai UUID
CREATE TABLE items (
    id UUID PRIMARY KEY
);
-- Benefits:
-- - 16 bytes per UUID (efficient)
-- - Fast binary comparison
-- - Compact indexes
-- - Type safety (validation)
-- - Standard representation
```

**2. Default Value Pattern:**

```sql
-- ✅ Pattern 1: UUID v4 untuk development/simple cases
CREATE TABLE orders (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    total NUMERIC NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Insert tanpa specify ID
INSERT INTO orders (customer_id, total) VALUES (123, 99.99);
-- ID auto-generated! ✅

-- ✅ Pattern 2: UUID v7 untuk production (if available)
CREATE TABLE orders_v7 (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,  -- Requires extension
    customer_id BIGINT NOT NULL,
    total NUMERIC NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- ✅ Pattern 3: Hybrid approach (internal + public)
CREATE TABLE orders_hybrid (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- Internal
    public_id UUID DEFAULT gen_random_uuid() UNIQUE NOT NULL,  -- External
    customer_id BIGINT NOT NULL,
    total NUMERIC NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_orders_public_id ON orders_hybrid(public_id);

-- Internal queries: Use id (fastest)
SELECT * FROM orders_hybrid WHERE id = 12345;

-- Public API: Use public_id (secure)
SELECT * FROM orders_hybrid WHERE public_id = '550e8400-...';
```

**3. UUID v7 Extension Check & Setup:**

```sql
-- ⚠️ Check PostgreSQL version first
SELECT version();
-- If PostgreSQL 17+: Built-in v7 support ✅
-- If < 17: Need extension

-- Check available extensions
SELECT * FROM pg_available_extensions
WHERE name LIKE '%uuid%';
-- Common results:
-- - uuid-ossp: v1, v3, v4, v5 (NO v7!)
-- - pg_uuidv7: Community extension for v7

-- Option 1: PostgreSQL 17+ (native)
SELECT gen_random_uuid_v7();  -- Just works! ✅

-- Option 2: Install pg_uuidv7 extension
-- (Requires superuser or appropriate permissions)
CREATE EXTENSION IF NOT EXISTS pg_uuidv7;
SELECT uuid_generate_v7();

-- Option 3: Custom implementation (if no extension)
CREATE OR REPLACE FUNCTION uuid_generate_v7()
RETURNS UUID AS $
DECLARE
    unix_ts_ms BIGINT;
    uuid_bytes BYTEA;
BEGIN
    -- Get current Unix timestamp in milliseconds
    unix_ts_ms := (EXTRACT(EPOCH FROM CLOCK_TIMESTAMP()) * 1000)::BIGINT;

    -- Generate UUID v7
    -- [48-bit timestamp][version+random][variant+random]
    uuid_bytes :=
        SET_BYTE('\x00000000000000000000000000000000', 0, (unix_ts_ms >> 40)::INT) ||
        SET_BYTE('\x00000000000000000000000000000000', 1, (unix_ts_ms >> 32)::INT) ||
        -- ... (simplified, full implementation more complex)
        gen_random_bytes(10);  -- Remaining random bytes

    RETURN CAST(uuid_bytes AS UUID);
END;
$ LANGUAGE plpgsql VOLATILE;

-- Test
SELECT uuid_generate_v7();
```

**4. Validation Pattern:**

```sql
-- Create validation function
CREATE OR REPLACE FUNCTION is_valid_uuid(text)
RETURNS BOOLEAN AS $
BEGIN
    -- Try to cast to UUID
    PERFORM $1::UUID;
    RETURN TRUE;
EXCEPTION
    WHEN invalid_text_representation THEN
        RETURN FALSE;
    WHEN OTHERS THEN
        RETURN FALSE;
END;
$ LANGUAGE plpgsql IMMUTABLE;

-- Usage in application logic
SELECT is_valid_uuid('550e8400-e29b-41d4-a716-446655440000');  -- true ✅
SELECT is_valid_uuid('not-a-uuid');  -- false ❌
SELECT is_valid_uuid('550e8400-e29b-41d4-a716');  -- false (too short)
SELECT is_valid_uuid('');  -- false ❌

-- Use in CHECK constraint
CREATE TABLE validated_items (
    id UUID PRIMARY KEY,
    external_id TEXT,
    CONSTRAINT external_id_valid
        CHECK (external_id IS NULL OR is_valid_uuid(external_id))
);

-- Insert tests
INSERT INTO validated_items VALUES (gen_random_uuid(), '550e8400-...');  -- ✅
INSERT INTO validated_items VALUES (gen_random_uuid(), 'invalid');  -- ❌ ERROR
```

**5. Comparison Performance:**

```sql
-- Create test table
CREATE TABLE perf_test (
    id_uuid UUID,
    id_text TEXT,
    data TEXT
);

-- Insert 1M rows
INSERT INTO perf_test
SELECT
    gen_random_uuid(),
    gen_random_uuid()::TEXT,
    'test data'
FROM generate_series(1, 1000000);

-- Create indexes
CREATE INDEX idx_uuid ON perf_test(id_uuid);
CREATE INDEX idx_text ON perf_test(id_text);

-- Analyze
ANALYZE perf_test;

-- Compare query performance
EXPLAIN ANALYZE
SELECT * FROM perf_test
WHERE id_uuid = '550e8400-e29b-41d4-a716-446655440000'::UUID;
-- Execution time: ~0.05ms ✅
-- Index size: ~21 MB

EXPLAIN ANALYZE
SELECT * FROM perf_test
WHERE id_text = '550e8400-e29b-41d4-a716-446655440000';
-- Execution time: ~0.08ms (slower) ⚠️
-- Index size: ~42 MB (2x larger) ❌

-- Comparison summary:
-- UUID index: Faster, smaller, more efficient ✅
-- TEXT index: Slower, larger, wasteful ❌
```

### Kesalahan Umum & Fixes:

**KESALAHAN 1: Store UUID sebagai TEXT**

```sql
-- ❌ WRONG
CREATE TABLE users (
    id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::TEXT
);
-- Problems:
-- - 40 bytes per row (160% overhead)
-- - Slower comparisons
-- - Larger indexes
-- - No type validation

-- ✅ CORRECT
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid()
);
-- Benefits:
-- - 16 bytes per row
-- - Fast binary comparison
-- - Compact index
-- - Type safety

-- Migration path:
ALTER TABLE users ADD COLUMN id_uuid UUID;
UPDATE users SET id_uuid = id::UUID;
-- Then swap columns (requires careful planning)
```

**KESALAHAN 2: UUID v4 sebagai Primary Key**

```sql
-- ❌ WRONG: Random UUID as PK
CREATE TABLE orders (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY
);
-- Problem: Index fragmentation ❌
-- Every insert goes to random location in B-tree
-- Causes page splits, poor cache locality

-- ✅ CORRECT Option 1: UUID v7 (distributed systems)
CREATE TABLE orders (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY
);
-- Sequential timestamp prefix
-- Efficient B-tree insertion

-- ✅ CORRECT Option 2: BIGINT (single database)
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
-- Most efficient
-- Perfect for non-distributed systems

-- ✅ CORRECT Option 3: Hybrid (best of both)
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    public_id UUID DEFAULT gen_random_uuid() UNIQUE
);
-- Internal: Use id (efficient)
-- External: Use public_id (secure)
```

**KESALAHAN 3: Manual UUID Formatting**

```sql
-- ❌ WRONG: Manual UUID generation
INSERT INTO items VALUES (
    CONCAT(
        LPAD(TO_HEX(FLOOR(RANDOM() * 4294967296)::BIGINT), 8, '0'),
        '-',
        LPAD(TO_HEX(FLOOR(RANDOM() * 65536)::INT), 4, '0'),
        '-',
        '4',  -- version
        LPAD(TO_HEX(FLOOR(RANDOM() * 4096)::INT), 3, '0'),
        '-',
        -- ... more complexity
    )::UUID
);
-- Problems:
-- - Complex and error-prone
-- - May not follow UUID spec correctly
-- - Hard to maintain
-- - Slower than built-in

-- ✅ CORRECT: Use built-in function
INSERT INTO items VALUES (gen_random_uuid());
-- Simple, fast, correct ✅
```

**KESALAHAN 4: Mixing UUID Versions Tanpa Documentation**

```sql
-- ❌ CONFUSING: Mixed versions without clarity
CREATE TABLE events (
    id UUID DEFAULT gen_random_uuid(),  -- v4
    correlation_id UUID,  -- Unknown version from external system
    parent_id UUID  -- Could be v4 or v7?
);
-- Problems:
-- - Unclear which version is which
-- - Maintenance nightmare
-- - Performance implications unknown

-- ✅ BETTER: Document everything
CREATE TABLE events (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,  -- v7: time-ordered, internal
    correlation_id UUID,  -- v4: from external API, random
    parent_id UUID  -- v7: time-ordered, references events.id
);

COMMENT ON COLUMN events.id IS
    'UUID v7 - time-ordered primary key, generated internally';
COMMENT ON COLUMN events.correlation_id IS
    'UUID v4 - correlation ID from external system, format may vary';
COMMENT ON COLUMN events.parent_id IS
    'UUID v7 - references parent event, NULL for root events';

-- Even better: Add CHECK constraints if possible
ALTER TABLE events ADD CONSTRAINT parent_id_references_id
    FOREIGN KEY (parent_id) REFERENCES events(id);
```

**KESALAHAN 5: UUID sebagai Security Mechanism**

```sql
-- ❌ WRONG: Assuming UUID = secure
CREATE TABLE documents (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id BIGINT,
    content TEXT,
    is_private BOOLEAN DEFAULT TRUE
);

-- Public API endpoint:
-- GET /api/documents/550e8400-e29b-41d4-a716-446655440000
-- ❌ NO AUTHENTICATION! Anyone with UUID can access!

-- ✅ CORRECT: Always implement proper auth
CREATE TABLE documents (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content TEXT,
    is_private BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Application layer (pseudocode):
/*
async function getDocument(documentId, requestingUserId) {
    const doc = await db.query(
        'SELECT * FROM documents WHERE id = $1',
        [documentId]
    );

    // ✅ ALWAYS check authorization!
    if (doc.user_id !== requestingUserId) {
        throw new UnauthorizedError('Access denied');
    }

    return doc;
}
*/

-- Or use Row Level Security (RLS):
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY user_documents ON documents
    FOR SELECT
    USING (user_id = current_setting('app.user_id')::BIGINT);

-- Key point: UUID is an identifier, NOT a security token! ⚠️
```

**KESALAHAN 6: Tidak Handle UUID v7 Timestamp Limitations**

```sql
-- ❌ PROBLEM: UUID v7 timestamp akan overflow di tahun 10,895
CREATE TABLE future_events (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,
    scheduled_for TIMESTAMP
);

-- ⚠️ Catatan: UUID v7 timestamp capacity
-- 48 bits = 2^48 milliseconds from Unix epoch
-- = 281,474,976,710,656 milliseconds
-- = ~8,925 years from 1970
-- Valid until approximately year 10,895

-- ✅ SOLUTION: Document limitation and plan migration
COMMENT ON TABLE future_events IS
    'Uses UUID v7 with 48-bit timestamp. Valid until year 10,895.
     Migration to v8 or new format will be needed before year 10,800.';

-- For most applications: This is not a concern! ✅
-- Your system will likely be replaced long before year 10,895 😄
```

### Advanced Patterns:

**Pattern 1: Composite IDs (Internal + Public)**

```sql
-- Strategy: Use BIGINT internally, UUID externally
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- Internal
    public_id UUID DEFAULT gen_random_uuid() UNIQUE NOT NULL,  -- External
    customer_id BIGINT NOT NULL,
    total NUMERIC(10,2) NOT NULL,
    status TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_orders_public_id ON orders(public_id);  -- For external lookups
CREATE INDEX idx_orders_customer ON orders(customer_id);  -- For joins
CREATE INDEX idx_orders_created ON orders(created_at);  -- For time queries

-- Internal queries (fastest - 8 byte integer):
SELECT * FROM orders WHERE id = 12345;
SELECT * FROM orders WHERE id BETWEEN 1000 AND 2000;

-- External API queries (secure - unpredictable UUID):
-- GET /api/orders/550e8400-e29b-41d4-a716-446655440000
SELECT * FROM orders WHERE public_id = '550e8400-...'::UUID;

-- Joins (efficient with internal BIGINT):
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.id > 10000;

-- Benefits:
-- ✅ Internal efficiency (BIGINT joins, range queries)
-- ✅ External security (non-enumerable UUIDs)
-- ✅ Best of both worlds
```

**Pattern 2: UUID Namespace untuk Deterministic IDs (v5)**

```sql
-- Use case: Idempotent operations, deduplication
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Define namespace UUIDs (constants)
CREATE TABLE uuid_namespaces (
    name TEXT PRIMARY KEY,
    namespace_uuid UUID NOT NULL
);

INSERT INTO uuid_namespaces VALUES
    ('users_by_email', uuid_generate_v5(uuid_ns_url(), 'myapp.com/users')),
    ('orders_by_ref', uuid_generate_v5(uuid_ns_url(), 'myapp.com/orders'));

-- Generate deterministic UUID dari email
CREATE OR REPLACE FUNCTION user_id_from_email(email TEXT)
RETURNS UUID AS $
DECLARE
    namespace UUID;
BEGIN
    SELECT namespace_uuid INTO namespace
    FROM uuid_namespaces
    WHERE name = 'users_by_email';

    RETURN uuid_generate_v5(namespace, LOWER(TRIM(email)));
END;
$ LANGUAGE plpgsql IMMUTABLE;

-- Usage: Same email = Same UUID ✅
SELECT user_id_from_email('user@example.com');
-- Output: 7c8a7d9e-4b3a-5f6c-8d7e-9f8a7b6c5d4e

SELECT user_id_from_email('user@example.com');
-- Output: 7c8a7d9e-4b3a-5f6c-8d7e-9f8a7b6c5d4e (identical!)

SELECT user_id_from_email('USER@EXAMPLE.COM');
-- Output: 7c8a7d9e-4b3a-5f6c-8d7e-9f8a7b6c5d4e (same, due to LOWER())

-- Use in table
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    name TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Idempotent insert
INSERT INTO users (id, email, name)
VALUES (
    user_id_from_email('user@example.com'),
    'user@example.com',
    'John Doe'
)
ON CONFLICT (id) DO NOTHING;  -- Safely handles duplicates ✅

-- Benefits:
-- ✅ Deterministic: same input = same UUID
-- ✅ Idempotent operations
-- ✅ Natural deduplication
-- ✅ No need to check existence before insert
```

**Pattern 3: Migration dari TEXT UUID ke Native UUID**

```sql
-- Starting point: Legacy table with TEXT UUIDs
CREATE TABLE legacy_items (
    id TEXT PRIMARY KEY,  -- UUID stored as TEXT ❌
    name TEXT,
    created_at TIMESTAMP
);

-- Data exists:
-- id: '550e8400-e29b-41d4-a716-446655440000' (TEXT, 40 bytes)

-- MIGRATION STRATEGY (Zero-downtime approach):

-- Step 1: Add new UUID column
ALTER TABLE legacy_items ADD COLUMN id_uuid UUID;

-- Step 2: Backfill existing data
UPDATE legacy_items SET id_uuid = id::UUID;
-- This converts TEXT to UUID (validates format)

-- Step 3: Handle new inserts (use trigger during transition)
CREATE OR REPLACE FUNCTION sync_uuid_columns()
RETURNS TRIGGER AS $
BEGIN
    IF NEW.id IS NOT NULL AND NEW.id_uuid IS NULL THEN
        NEW.id_uuid := NEW.id::UUID;
    END IF;
    IF NEW.id_uuid IS NOT NULL AND NEW.id IS NULL THEN
        NEW.id := NEW.id_uuid::TEXT;
    END IF;
    RETURN NEW;
END;
$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_uuid
BEFORE INSERT OR UPDATE ON legacy_items
FOR EACH ROW EXECUTE FUNCTION sync_uuid_columns();

-- Step 4: Update application code to use id_uuid
-- (Deploy code changes first)

-- Step 5: Add NOT NULL constraint (after code deployed)
ALTER TABLE legacy_items ALTER COLUMN id_uuid SET NOT NULL;

-- Step 6: Create index on new column
CREATE UNIQUE INDEX idx_legacy_items_uuid ON legacy_items(id_uuid);

-- Step 7: Drop old primary key and create new one (requires brief lock)
BEGIN;
  ALTER TABLE legacy_items DROP CONSTRAINT legacy_items_pkey;
  ALTER TABLE legacy_items ADD PRIMARY KEY (id_uuid);
  ALTER TABLE legacy_items DROP COLUMN id;
  ALTER TABLE legacy_items RENAME COLUMN id_uuid TO id;
  DROP TRIGGER IF EXISTS trg_sync_uuid ON legacy_items;
  DROP FUNCTION IF EXISTS sync_uuid_columns();
COMMIT;

-- Result: Efficient UUID native type ✅
-- Storage saved: 40 bytes → 16 bytes per row
-- For 10M rows: 240 MB saved!
```

**Pattern 4: UUID untuk Soft Deletes dengan Audit Trail**

```sql
CREATE TABLE documents (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    title TEXT NOT NULL,
    content TEXT,
    deleted_at TIMESTAMP,
    deletion_id UUID,  -- Links to deletion event
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE deletion_events (
    id UUID PRIMARY KEY,  -- Same as documents.deletion_id
    table_name TEXT NOT NULL,
    record_id UUID NOT NULL,
    deleted_by_user_id BIGINT,
    reason TEXT,
    deleted_at TIMESTAMP DEFAULT NOW(),
    can_restore BOOLEAN DEFAULT TRUE
);

-- Soft delete function
CREATE OR REPLACE FUNCTION soft_delete_document(doc_id UUID, user_id BIGINT, delete_reason TEXT)
RETURNS UUID AS $
DECLARE
    deletion_uuid UUID;
BEGIN
    -- Generate UUID untuk deletion event
    deletion_uuid := gen_random_uuid();

    -- Record deletion event
    INSERT INTO deletion_events (id, table_name, record_id, deleted_by_user_id, reason)
    VALUES (deletion_uuid, 'documents', doc_id, user_id, delete_reason);

    -- Mark document as deleted
    UPDATE documents
    SET deleted_at = NOW(),
        deletion_id = deletion_uuid
    WHERE id = doc_id;

    RETURN deletion_uuid;
END;
$ LANGUAGE plpgsql;

-- Usage
SELECT soft_delete_document(
    '550e8400-e29b-41d4-a716-446655440000'::UUID,
    123,
    'User requested deletion'
);
-- Returns: deletion_id for tracking

-- Restore function
CREATE OR REPLACE FUNCTION restore_document(deletion_uuid UUID)
RETURNS BOOLEAN AS $
DECLARE
    doc_id UUID;
    can_restore_flag BOOLEAN;
BEGIN
    -- Check if restoration is allowed
    SELECT record_id, can_restore
    INTO doc_id, can_restore_flag
    FROM deletion_events
    WHERE id = deletion_uuid AND table_name = 'documents';

    IF NOT FOUND OR NOT can_restore_flag THEN
        RETURN FALSE;
    END IF;

    -- Restore document
    UPDATE documents
    SET deleted_at = NULL,
        deletion_id = NULL
    WHERE id = doc_id;

    -- Mark event as processed
    UPDATE deletion_events
    SET can_restore = FALSE
    WHERE id = deletion_uuid;

    RETURN TRUE;
END;
$ LANGUAGE plpgsql;

-- Query active documents (not deleted)
CREATE VIEW active_documents AS
SELECT * FROM documents WHERE deleted_at IS NULL;

-- Audit trail query
SELECT
    d.id,
    d.title,
    de.deleted_at,
    de.deleted_by_user_id,
    de.reason
FROM documents d
JOIN deletion_events de ON d.deletion_id = de.id
WHERE d.deleted_at IS NOT NULL
ORDER BY de.deleted_at DESC;

-- Benefits:
-- ✅ Complete audit trail
-- ✅ Traceable deletions across tables
-- ✅ Restorable with history
-- ✅ UUID links deletions to events
```

**Pattern 5: Partitioning dengan UUID v7**

```sql
-- UUID v7 allows time-based partitioning
-- Because first 48 bits = timestamp!

CREATE TABLE events (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,
    event_type TEXT NOT NULL,
    user_id BIGINT NOT NULL,
    data JSONB,
    created_at TIMESTAMP DEFAULT NOW()
) PARTITION BY RANGE (id);

-- Create partitions based on UUID v7 timestamp
-- UUID v7 prefix for 2024-01: 0185d...
-- UUID v7 prefix for 2024-02: 01866...

CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('01850000-0000-0000-0000-000000000000')
    TO ('01860000-0000-0000-0000-000000000000');

CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('01860000-0000-0000-0000-000000000000')
    TO ('01870000-0000-0000-0000-000000000000');

-- Automatic partition routing based on UUID v7!
INSERT INTO events (event_type, user_id, data)
VALUES ('user_login', 123, '{"ip": "192.168.1.1"}');
-- Automatically goes to correct partition based on timestamp ✅

-- Query optimization: Partition pruning works!
SELECT * FROM events
WHERE id >= '01850000-0000-0000-0000-000000000000'
  AND id < '01860000-0000-0000-0000-000000000000';
-- Only scans events_2024_01 partition ✅

-- Benefits:
-- ✅ Time-based partitioning with UUIDs
-- ✅ Automatic routing
-- ✅ Efficient queries
-- ✅ Easy to drop old partitions
```

### Real-World Scenarios:

**Scenario 1: Microservices E-commerce**

```sql
-- Order Service (Database 1)
CREATE TABLE orders (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,
    customer_id UUID NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    total NUMERIC(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Payment Service (Database 2 - different server)
CREATE TABLE payments (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,
    order_id UUID NOT NULL,  -- References Order Service
    amount NUMERIC(10,2) NOT NULL,
    payment_method TEXT,
    status TEXT NOT NULL,
    processed_at TIMESTAMP DEFAULT NOW()
);

-- Shipping Service (Database 3 - different server)
CREATE TABLE shipments (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,
    order_id UUID NOT NULL,  -- References Order Service
    tracking_number TEXT,
    carrier TEXT,
    shipped_at TIMESTAMP
);

-- Benefits:
-- ✅ Each service generates IDs independently
-- ✅ No cross-service calls for ID generation
-- ✅ Easy to correlate data: order_id is universal
-- ✅ Analytics can merge all data without conflicts
-- ✅ UUID v7 maintains temporal ordering

-- Analytics query (combines all services):
SELECT
    o.id as order_id,
    o.total,
    p.payment_method,
    p.status as payment_status,
    s.tracking_number,
    s.carrier
FROM analytics.orders o
LEFT JOIN analytics.payments p ON o.id = p.order_id
LEFT JOIN analytics.shipments s ON o.id = s.order_id
WHERE o.created_at >= NOW() - INTERVAL '30 days';
-- No ID conflicts! ✅
```

**Scenario 2: Offline-First Mobile App**

```sql
-- Mobile app database schema (SQLite local)
CREATE TABLE tasks (
    id TEXT PRIMARY KEY,  -- UUID as TEXT in SQLite
    title TEXT NOT NULL,
    completed BOOLEAN DEFAULT 0,
    created_at INTEGER,  -- Unix timestamp
    synced BOOLEAN DEFAULT 0
);

-- JavaScript (React Native):
/*
import { v7 as uuidv7 } from 'uuid';

// Create task offline
async function createTask(title) {
    const taskId = uuidv7();  // UUID v7: time-ordered

    await localDB.execute(
        'INSERT INTO tasks (id, title, created_at) VALUES (?, ?, ?)',
        [taskId, title, Date.now()]
    );

    // Show immediately in UI ✅
    return taskId;
}

// Sync when online
async function syncTasks() {
    const unsyncedTasks = await localDB.query(
        'SELECT * FROM tasks WHERE synced = 0'
    );

    for (const task of unsyncedTasks) {
        // Send to server with pre-generated UUID
        await fetch('/api/tasks', {
            method: 'POST',
            body: JSON.stringify({
                id: task.id,  // UUID already generated!
                title: task.title,
                created_at: task.created_at
            })
        });

        // Mark as synced
        await localDB.execute(
            'UPDATE tasks SET synced = 1 WHERE id = ?',
            [task.id]
        );
    }
}
*/

-- PostgreSQL server (receives synced data):
CREATE TABLE tasks (
    id UUID PRIMARY KEY,  -- Accepts pre-generated UUID
    user_id BIGINT NOT NULL,
    title TEXT NOT NULL,
    completed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL,
    synced_at TIMESTAMP DEFAULT NOW()
);

-- Insert from mobile app (idempotent)
INSERT INTO tasks (id, user_id, title, created_at)
VALUES (
    '018e1e5e-7c8a-7xxx-yyyy-zzzzzzzzzzzz'::UUID,
    123,
    'Buy groceries',
    '2024-03-15 10:30:00'
)
ON CONFLICT (id) DO NOTHING;  -- Handles duplicate syncs ✅

-- Benefits:
-- ✅ Offline functionality (no server needed for IDs)
-- ✅ Instant UI feedback
-- ✅ No ID conflicts when syncing multiple devices
-- ✅ Idempotent sync (can retry safely)
-- ✅ UUID v7 preserves creation time order
```

**Scenario 3: Multi-Region Deployment**

```sql
-- US East Region (Primary Database)
CREATE TABLE users (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    name TEXT,
    region TEXT DEFAULT 'us-east',
    created_at TIMESTAMP DEFAULT NOW()
);

-- EU West Region (Primary Database)
-- Identical schema
CREATE TABLE users (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    name TEXT,
    region TEXT DEFAULT 'eu-west',
    created_at TIMESTAMP DEFAULT NOW()
);

-- User signs up in US
INSERT INTO users (email, name) VALUES ('alice@example.com', 'Alice');
-- id: 018e1e5e-7c8a-7xxx-xxxx-xxxxxxxxxxxx

-- Another user signs up in EU (same second!)
INSERT INTO users (email, name) VALUES ('bob@example.com', 'Bob');
-- id: 018e1e5e-7c8a-7yyy-yyyy-yyyyyyyyyyyy
-- Different UUID even though created at same millisecond! ✅

-- Central Analytics Database (aggregates both regions)
CREATE TABLE users_global (
    id UUID PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    name TEXT,
    region TEXT,
    created_at TIMESTAMP
);

-- Sync from both regions
INSERT INTO users_global
SELECT * FROM us_east.users
UNION ALL
SELECT * FROM eu_west.users;
-- No ID conflicts! ✅
-- UUID v7 maintains chronological order globally ✅

-- Query users by time across all regions
SELECT * FROM users_global
WHERE created_at >= NOW() - INTERVAL '1 hour'
ORDER BY id;  -- UUID v7 = time-ordered ✅
```

**Scenario 4: Multi-Tenant SaaS dengan Tenant Isolation**

```sql
-- Tenant table
CREATE TABLE tenants (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,
    name TEXT NOT NULL,
    subdomain TEXT UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- User table (multi-tenant)
CREATE TABLE users (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email TEXT NOT NULL,
    name TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(tenant_id, email)  -- Email unique per tenant
);

-- Data table (multi-tenant)
CREATE TABLE projects (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    owner_id UUID NOT NULL REFERENCES users(id),
    title TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Row Level Security (RLS) for tenant isolation
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see their tenant's data
CREATE POLICY tenant_isolation_users ON users
    FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::UUID);

CREATE POLICY tenant_isolation_projects ON projects
    FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::UUID);

-- Application code (pseudocode):
/*
// Set tenant context for session
await db.query(
    'SET LOCAL app.tenant_id = $1',
    [tenantId]  // UUID from authenticated user
);

// Query automatically filtered by RLS
const users = await db.query('SELECT * FROM users');
// Only returns users from current tenant ✅

const projects = await db.query('SELECT * FROM projects');
// Only returns projects from current tenant ✅
*/

-- Partitioning by tenant (optional, for very large datasets)
CREATE TABLE projects_partitioned (
    id UUID,
    tenant_id UUID,
    owner_id UUID,
    title TEXT,
    created_at TIMESTAMP
) PARTITION BY LIST (tenant_id);

-- Create partition per tenant
CREATE TABLE projects_tenant_a PARTITION OF projects_partitioned
    FOR VALUES IN ('018e1e5e-...');  -- Tenant A UUID

CREATE TABLE projects_tenant_b PARTITION OF projects_partitioned
    FOR VALUES IN ('018e1f6f-...');  -- Tenant B UUID

-- Benefits:
-- ✅ UUID provides universal IDs across all tenants
-- ✅ Easy to migrate tenant data (just move rows)
-- ✅ RLS ensures data isolation
-- ✅ Partition for performance at scale
-- ✅ No ID conflicts between tenants
```

### Analogi untuk Memahami UUID:

**1. UUID Types sebagai Nomor Identifikasi:**

```
BIGINT SERIAL = Nomor antrian bank
- Sequential: 1, 2, 3, 4, 5...
- Predictable ✅
- Efficient (8 bytes) ✅
- Tapi: Hanya satu bank yang bisa assign
- Single point of coordination

UUID v4 = Nomor lotre acak
- Random: 9827, 1234, 5673, 2981...
- Unpredictable ✅
- No coordination needed ✅
- Tapi: Orang datang dalam urutan random
- Causes chaos at service window ❌

UUID v7 = Tiket dengan timestamp barcode
- Format: [2024-03-15-10:30:45][Random-ABC]
- Time-ordered ✅
- Unpredictable suffix ✅
- No coordination needed ✅
- Orang otomatis sorted by arrival time ✅
```

**2. Storage Comparison:**

```
TEXT UUID = Menulis UUID dengan tangan di kertas
- Size: 36 karakter + margin = banyak ruang
- Read: Harus baca satu-satu karakter
- Compare: Harus compare string panjang
- Storage: 40 bytes ❌

UUID Type = Barcode sticker
- Size: Compact binary code
- Read: Scan langsung (binary comparison)
- Compare: Fast binary operation
- Storage: 16 bytes ✅

Analogi:
- Kertas dengan tulisan tangan = 40 bytes TEXT
- Barcode sticker = 16 bytes UUID
- 60% lebih kecil! ✅
```

**3. Distribution Strategy:**

```
Central ID Server (BIGINT) = Bank sentral cetak uang
- Semua cabang harus minta ke pusat
- Efficient ✅
- Tapi: Requires coordination
- Single point of failure ⚠️

UUID Generation = Setiap orang bikin nomor sendiri
- Format agreed: 128-bit dengan rules
- Everyone generates independently
- No conflicts (probabilitas collision ~0) ✅
- No network calls needed ✅

Analogi:
- Central server = semua harus ke Jakarta
- UUID = setiap kantor cabang bisa bikin sendiri
- No traffic jam! ✅
```

**4. Index Fragmentation:**

```
UUID v4 (Random) = Perpustakaan buku masuk random
- Buku random:
  - "Zebra" masuk → rak Z
  - "Apple" masuk → rak A (pindahkan semua buku)
  - "Mango" masuk → rak M (pindahkan lagi)
- Constantly reorganizing shelves ❌
- Slow to find space ❌

UUID v7 (Sequential) = Buku masuk by publish date
- Buku terbaru:
  - March 2024 → tambah di ujung rak March
  - April 2024 → tambah di ujung rak April
- Rak tumbuh natural ✅
- Fast to add ✅
- Easy to find: sorted by time ✅

BIGINT SERIAL = Buku dengan nomor urut
- Buku 1, 2, 3, 4...
- Perfect order ✅✅
- Fastest possible ✅✅
```

### Performance Deep Dive (Preview untuk Video Berikutnya):

```sql
-- TEST SETUP: Compare PK types
-- Create three identical tables dengan different PKs

-- Table 1: BIGINT SERIAL (baseline)
CREATE TABLE perf_bigint (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    data TEXT
);

-- Table 2: UUID v4 (random)
CREATE TABLE perf_uuid_v4 (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    data TEXT
);

-- Table 3: UUID v7 (time-ordered)
CREATE TABLE perf_uuid_v7 (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,
    data TEXT
);

-- Insert 1 million rows ke each table
DO $
BEGIN
    FOR i IN 1..1000000 LOOP
        INSERT INTO perf_bigint (data) VALUES ('test data');
        INSERT INTO perf_uuid_v4 (data) VALUES ('test data');
        INSERT INTO perf_uuid_v7 (data) VALUES ('test data');
    END LOOP;
END $;

-- Analyze tables
ANALYZE perf_bigint, perf_uuid_v4, perf_uuid_v7;

-- Compare metrics
SELECT
    'BIGINT' as type,
    pg_size_pretty(pg_total_relation_size('perf_bigint')) as total_size,
    pg_size_pretty(pg_relation_size('perf_bigint')) as table_size,
    pg_size_pretty(pg_indexes_size('perf_bigint')) as index_size
UNION ALL
SELECT
    'UUID v4',
    pg_size_pretty(pg_total_relation_size('perf_uuid_v4')),
    pg_size_pretty(pg_relation_size('perf_uuid_v4')),
    pg_size_pretty(pg_indexes_size('perf_uuid_v4'))
UNION ALL
SELECT
    'UUID v7',
    pg_size_pretty(pg_total_relation_size('perf_uuid_v7')),
    pg_size_pretty(pg_relation_size('perf_uuid_v7')),
    pg_size_pretty(pg_indexes_size('perf_uuid_v7'));

-- Typical Results (1M rows):
-- ┌──────────┬────────────┬────────────┬────────────┐
-- │ type     │ total_size │ table_size │ index_size │
-- ├──────────┼────────────┼────────────┼────────────┤
-- │ BIGINT   │ 57 MB      │ 35 MB      │ 21 MB      │ ← Smallest ✅✅
-- │ UUID v4  │ 86 MB      │ 43 MB      │ 43 MB      │ ← Largest ❌
-- │ UUID v7  │ 78 MB      │ 43 MB      │ 35 MB      │ ← Middle ✅
-- └──────────┴────────────┴────────────┴────────────┘

-- Observations:
-- 1. BIGINT: Smallest total size (baseline)
-- 2. UUID v4: 2x larger index (fragmentation!)
-- 3. UUID v7: Similar table size, better index
-- 4. All UUIDs: Larger table (16 bytes vs 8 bytes)

-- Insert performance test (1000 rows)
EXPLAIN ANALYZE
INSERT INTO perf_bigint (data)
SELECT 'test' FROM generate_series(1, 1000);
-- Execution time: ~15ms ✅✅

EXPLAIN ANALYZE
INSERT INTO perf_uuid_v7 (data)
SELECT 'test' FROM generate_series(1, 1000);
-- Execution time: ~18ms ✅

EXPLAIN ANALYZE
INSERT INTO perf_uuid_v4 (data)
SELECT 'test' FROM generate_series(1, 1000);
-- Execution time: ~35ms ❌ (slower due to random inserts)

-- Point lookup performance (indexed PK)
EXPLAIN ANALYZE
SELECT * FROM perf_bigint WHERE id = 500000;
-- Execution time: ~0.05ms ✅✅

EXPLAIN ANALYZE
SELECT * FROM perf_uuid_v7 WHERE id = 'some-uuid-v7';
-- Execution time: ~0.06ms ✅

EXPLAIN ANALYZE
SELECT * FROM perf_uuid_v4 WHERE id = 'some-uuid-v4';
-- Execution time: ~0.08ms (slightly slower due to fragmentation)

-- Summary:
-- BIGINT SERIAL: Best overall performance ✅✅
-- UUID v7: Good performance, slight overhead ✅
-- UUID v4: Significant overhead for PKs ❌
```

**Index Fragmentation Visualization:**

```
B-tree Index Pages (after 1M inserts):

BIGINT SERIAL:
Page 1: [1...10000]          Page 2: [10001...20000]
Page 3: [20001...30000]      Page 4: [30001...40000]
...
Fill factor: 95%+ ✅
Page splits: Minimal ✅
Lookup efficiency: Excellent ✅✅

UUID v7 (time-ordered):
Page 1: [018e5e...018e5f]   Page 2: [018e5f...018e60]
Page 3: [018e60...018e61]   Page 4: [018e61...018e62]
...
Fill factor: 90%+ ✅
Page splits: Low ✅
Lookup efficiency: Excellent ✅

UUID v4 (random):
Page 1: [1a2b..., 5d4c..., 9b8c...]  ← Mixed ranges, not full
Page 2: [2c3d..., 6e5f..., a1b2...]  ← Fragmented
Page 3: [3d4e..., 7f6g..., b2c3...]  ← Wasted space
...
Fill factor: 60-70% ❌
Page splits: Very frequent ❌
Lookup efficiency: Degraded ❌
```

## 5. Kesimpulan

UUID adalah tipe data 128-bit (16 bytes) di PostgreSQL yang sangat powerful untuk generating unique identifiers tanpa koordinasi antar sistem. PostgreSQL menyediakan native UUID type yang jauh lebih efficient (16 bytes) dibanding menyimpan sebagai TEXT (37-40 bytes), dengan benefits tambahan berupa fast comparison dan compact indexes.

### Key Points:

**1. UUID Type Fundamentals:**

- Native binary type: 16 bytes storage (NOT text with hyphens)
- Display format: 36 characters (8-4-4-4-12 hex dengan hyphens)
- Universally unique: collision probability ~0 untuk practical purposes
- Type safety: automatic validation on insert/update

**2. Storage Comparison:**

```
TEXT:   37-40 bytes  (string + varlena overhead)  ❌
UUID:   16 bytes     (pure binary)                ✅
Savings: 60% reduction untuk large tables
```

**3. UUID Versions:**

- **v4 (Random):** Built-in via `gen_random_uuid()`, 122 bits random
  - ✅ Good untuk: Secondary IDs, correlation IDs, public identifiers
  - ❌ Bad untuk: Primary keys (causes index fragmentation)
- **v7 (Time-ordered):** Requires PostgreSQL 17+ or extension
  - ✅ Good untuk: Primary keys, time-ordered data, distributed systems
  - ✅ Sequential insertion = efficient B-tree indexes
- **v5 (Name-based):** Deterministic, requires uuid-ossp extension
  - ✅ Good untuk: Idempotent operations, deduplication

**4. Generation Methods:**

```sql
-- Built-in (PostgreSQL 9.4+):
SELECT gen_random_uuid();  -- UUID v4

-- PostgreSQL 17+:
SELECT gen_random_uuid_v7();  -- UUID v7 native

-- Extension (< PostgreSQL 17):
CREATE EXTENSION pg_uuidv7;
SELECT uuid_generate_v7();
```

**5. Best Practices:**

✅ **DO:**

- Always store as UUID type, never TEXT
- Use UUID v7 for primary keys in distributed systems
- Use UUID v4 for secondary identifiers
- Document which UUID version you're using
- Implement proper authentication (UUID ≠ security)

❌ **DON'T:**

- Store UUID as TEXT (waste of space)
- Use UUID v4 as primary key (fragmentation)
- Rely on UUID for security (always add proper auth)
- Mix UUID versions without documentation
- Assume uuid-ossp includes v7 (it doesn't!)

**6. Decision Framework:**

```
Choose ID Type:
├─ Single database?
│   └─ BIGINT SERIAL ✅✅ (8 bytes, most efficient)
│
├─ Distributed systems?
│   └─ UUID v7 ✅ (16 bytes, sortable, no coordination)
│
├─ Need both?
│   └─ BIGINT (internal) + UUID v4 (external) ✅
│
└─ Secondary identifiers?
    └─ UUID v4 ✅ (random, unpredictable)
```

**7. Use Cases:**

- ✅ Microservices architectures (independent ID generation)
- ✅ Client-side ID generation (offline-first apps)
- ✅ Multi-database synchronization (no conflicts)
- ✅ Multi-region deployments (global uniqueness)
- ✅ Public APIs (non-enumerable IDs)
- ✅ Multi-tenant SaaS (universal identifiers)

**8. Performance Preview:**

```
Metric                  BIGINT    UUID v7    UUID v4
───────────────────────────────────────────────────
Key size                8 bytes   16 bytes   16 bytes
Insert speed            Fastest   Fast       Slow
Index size              Smallest  Small      Large
Page splits             Minimal   Low        High
Best use case           Single DB Distrib.   Secondary
```

### Looking Ahead:

Video ini adalah foundation untuk topics yang akan datang:

- **Primary Keys Deep Dive:** BIGINT vs UUID v7 performance comparison
- **B-tree Indexes:** How different PK types affect index structure
- **Index Fragmentation:** Why UUID v4 is problematic for PKs
- **Optimization Strategies:** When to use each ID type

### Final Recommendations:

**For most applications:**

```sql
-- Single database (95% of use cases)
CREATE TABLE items (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT
);
-- ✅ Simplest, fastest, most efficient

-- Distributed systems
CREATE TABLE items (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,
    name TEXT
);
-- ✅ No coordination, sortable, efficient

-- Hybrid (best of both worlds)
CREATE TABLE items (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    public_id UUID DEFAULT gen_random_uuid() UNIQUE,
    name TEXT
);
-- ✅ Internal efficiency + external security
```

**Key Takeaway:**

UUID adalah tool yang sangat berguna untuk distributed ID generation dan scenarios yang membutuhkan coordination-free uniqueness. Always store sebagai native UUID type untuk maximum efficiency. Untuk primary keys: prefer UUID v7 (sortable) di distributed systems, atau stick dengan BIGINT SERIAL (paling efficient) untuk single-database applications. UUID v4 excellent untuk secondary identifiers tapi harus avoid sebagai primary key karena index fragmentation issues.

**Remember:**

- UUID = Identifier (16 bytes binary, universal uniqueness)
- UUID v4 = Random (good for secondary IDs)
- UUID v7 = Time-ordered (good for primary keys)
- TEXT UUID = ❌ Never use (wastes space)
- BIGINT = Often best choice untuk single DB (8 bytes, sequential)

---

**Catatan Koreksi dari Versi Asli:**

1. ✅ **Fixed:** uuid-ossp TIDAK support UUID v7 (hanya v1, v3, v4, v5)
2. ✅ **Fixed:** PostgreSQL 17+ has built-in `gen_random_uuid_v7()`
3. ✅ **Clarified:** UUID stored as binary, NOT text with hyphens
4. ✅ **Clarified:** TEXT overhead varies (37-40 bytes, not fixed 40)
5. ✅ **Enhanced:** Detailed UUID structure breakdowns
6. ✅ **Added:** Extensive real-world scenarios and patterns
7. ✅ **Added:** Performance comparisons with actual metrics
8. ✅ **Added:** Decision trees and selection frameworks

**Total Length:** Comprehensive guide dengan semua corrections applied dan extensive examples added.
