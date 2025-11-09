# 📝 UUID sebagai Tipe Data di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas UUID (Universally Unique Identifier) sebagai tipe data di PostgreSQL, bukan sebagai primary key (yang akan dibahas di video terpisah). UUID adalah identifier 128-bit yang sangat unik dan berguna untuk generating IDs tanpa koordinasi antar sistem. Video menjelaskan keuntungan menyimpan UUID sebagai tipe data native (16 bytes) dibanding TEXT (40+ bytes), berbagai versi UUID (terutama v4 random dan v7 sortable), dan cara generate UUID di PostgreSQL.

## 2. Konsep Utama

### a. UUID sebagai Tipe Data

UUID di PostgreSQL adalah tipe data native yang menyimpan 128-bit identifier dengan format standar.

**Karakteristik UUID:**

- Fixed-size: 16 bytes storage
- Format: 8-4-4-4-12 hex digits (display format)
- Universally unique (collision probability sangat rendah)
- Native type dengan optimasi khusus

```sql
-- Membuat table dengan UUID column
CREATE TABLE uuid_example (
    uuid_value UUID
);

-- Insert UUID sebagai string (auto-converted)
INSERT INTO uuid_example VALUES ('550e8400-e29b-41d4-a716-446655440000');

-- View data
SELECT * FROM uuid_example;
-- Output: 550e8400-e29b-41d4-a716-446655440000

-- Cek tipe data
SELECT pg_typeof(uuid_value) FROM uuid_example;
-- Output: uuid (bukan text!)

-- Cek storage size
SELECT pg_column_size(uuid_value) FROM uuid_example;
-- Output: 16 bytes ✅
```

**Format UUID:**

```
550e8400-e29b-41d4-a716-446655440000
│      │ │  │ │  │ │  │ │          │
└──┬──┘ │  │ │  │ │  │ └────┬────┘
   8    4  4  4  4      12 hex digits

Total: 36 characters (display)
Storage: 16 bytes (efficient!)
```

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
-- text_type_size: 40 bytes  ❌

-- Breakdown TEXT size:
-- - 36 bytes untuk string '550e8400-e29b-41d4-a716-446655440000'
-- - 4 bytes untuk TEXT overhead (varlena header)
-- Total: 40 bytes

-- UUID native: 16 bytes (raw 128-bit storage)
```

**Impact pada Large Tables:**

```sql
-- Example: 10 million users
CREATE TABLE users_text (
    id TEXT  -- UUID sebagai TEXT: 40 bytes
);
-- Total: 10M × 40 bytes = 400 MB

CREATE TABLE users_uuid (
    id UUID  -- UUID sebagai UUID: 16 bytes
);
-- Total: 10M × 16 bytes = 160 MB

-- Savings: 240 MB (60% reduction!) ✅
-- Plus: Faster comparison dan indexing
```

### c. Universally Unique: Collision Probability

UUID dirancang untuk uniqueness tanpa koordinasi.

**Collision Probability:**

```
UUID space: 2^128 = 340,282,366,920,938,463,463,374,607,431,768,211,456

Probability collision dengan 1 trillion UUIDs:
~ 1 in 100 billion

More likely to:
- Get hit by lightning 40+ times
- Win lottery multiple times
- Find a specific grain of sand on all Earth's beaches

Conclusion: Effectively zero untuk practical purposes ✅
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
```

### d. Use Cases untuk UUID

UUID sangat berguna untuk specific scenarios.

#### ✅ Use Case 1: Distributed Systems

```sql
-- Microservices architecture
-- Service A: Order Service
CREATE TABLE orders (
    id UUID DEFAULT gen_random_uuid(),
    customer_id UUID,
    total NUMERIC
);

-- Service B: Shipping Service
CREATE TABLE shipments (
    id UUID DEFAULT gen_random_uuid(),
    order_id UUID,  -- Reference ke order dari Service A
    tracking_number TEXT
);

-- Keuntungan:
-- - Generate ID independently di setiap service
-- - No central ID coordinator needed
-- - Easy to merge/sync data antar services
```

#### ✅ Use Case 2: Client-Side Generation

```sql
-- Frontend generates ID sebelum server insert
-- JavaScript:
const newId = crypto.randomUUID();
// '550e8400-e29b-41d4-a716-446655440000'

// Send to server
fetch('/api/items', {
    method: 'POST',
    body: JSON.stringify({
        id: newId,
        name: 'New Item'
    })
});

-- PostgreSQL:
INSERT INTO items (id, name)
VALUES ('550e8400-e29b-41d4-a716-446655440000', 'New Item');

-- Keuntungan:
-- - Optimistic UI (show ID immediately)
-- - Offline-first apps
-- - No server roundtrip untuk get ID
```

#### ✅ Use Case 3: Multiple Databases Sync

```sql
-- Database 1 (Production)
INSERT INTO users (id, email) VALUES (uuid1, 'user@example.com');

-- Database 2 (Backup/Replica)
-- Same UUID, no conflict! ✅

-- Database 3 (Analytics)
-- Can merge data from multiple sources
-- No ID collision ✅
```

#### ⚠️ Use Case 4: Public IDs (dengan caveats)

```sql
-- Expose UUID ke public (URLs, APIs)
-- Advantage: Not sequential (harder to enumerate)
-- https://api.example.com/users/550e8400-e29b-41d4-a716-446655440000

-- vs sequential IDs:
-- https://api.example.com/users/12345
-- https://api.example.com/users/12346 (easy to guess!)

-- Tapi: UUID v7 has timestamp, jadi bisa sorted/predicted
-- Untuk security, tetap butuh proper authentication!
```

### e. Generating UUIDs di PostgreSQL

PostgreSQL bisa generate UUID, dengan limitations.

#### UUID v4: Random (Built-in)

```sql
-- Generate random UUID (version 4)
SELECT gen_random_uuid();
-- Output: 9b8c7e6d-5f4a-4b3c-9d8e-7f6a5b4c3d2e

-- Setiap call generates UUID baru
SELECT gen_random_uuid();
-- Output: 1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d (berbeda!)

-- Use sebagai default value
CREATE TABLE items (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    name TEXT
);

-- Insert tanpa specify ID
INSERT INTO items (name) VALUES ('Item 1');
-- ID auto-generated! ✅
```

**UUID v4 Characteristics:**

- Fully random (122 bits entropy)
- No sortable/sequential property
- ❌ BAD sebagai primary key (akan dibahas nanti)
- ✅ OK untuk secondary identifiers

#### UUID v7: Sortable (Extension Required)

```sql
-- UUID v7 TIDAK built-in, butuh extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
-- Atau gunakan extension lain yang support v7

-- UUID v7 format:
-- [timestamp][random]
-- 48-bit timestamp + 74-bit random

-- Keuntungan:
-- 1. Lexographically sortable (timestamp di depan)
-- 2. Bisa extract timestamp
-- 3. ✅ GOOD sebagai primary key!

-- Example structure (conceptual):
-- 018e1e5e-7c8a-7xxx-xxxx-xxxxxxxxxxxx
-- └──┬──┘
--    └─ Timestamp portion (sorted!)
```

**UUID v7 vs v4:**

```
UUID v4 (random):
- 9b8c7e6d-5f4a-4b3c-9d8e-7f6a5b4c3d2e
- 1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d
- c3d2e1f0-a9b8-c7d6-e5f4-a3b2c1d0e9f8
❌ Random order, fragments B-tree index

UUID v7 (timestamped):
- 018e1e5e-7c8a-7xxx-xxxx-xxxxxxxxxxxx (oldest)
- 018e1e5f-8d9b-7yyy-yyyy-yyyyyyyyyyyy
- 018e1e60-9eac-7zzz-zzzz-zzzzzzzzzzzz (newest)
✅ Sequential order, efficient B-tree index
```

### f. UUID Versions: Overview

Ada 8 versi UUID (saat recording), each dengan purpose berbeda.

| Version | Type               | Description                    | Use Case                               |
| ------- | ------------------ | ------------------------------ | -------------------------------------- |
| **v1**  | Time-based + MAC   | Timestamp + machine address    | Legacy, privacy concerns (MAC visible) |
| **v2**  | DCE Security       | Reserved, rarely used          | Specialized security                   |
| **v3**  | Name-based (MD5)   | MD5 hash of namespace + name   | Deterministic UUIDs                    |
| **v4**  | Random             | Fully random                   | General purpose, NOT for PKs           |
| **v5**  | Name-based (SHA-1) | SHA-1 hash of namespace + name | Deterministic, better than v3          |
| **v6**  | Reordered time     | Like v1 but sortable           | Improvement over v1                    |
| **v7**  | Unix timestamp     | Timestamp + random             | ✅ **Best for primary keys**           |
| **v8**  | Custom             | User-defined                   | Specialized needs                      |

**Key Differences:**

```
v4: Fully random
    ↓
❌ Bad for primary keys (random insertion → fragmented index)

v7: Timestamp prefix
    ↓
✅ Good for primary keys (sequential insertion → efficient index)
```

### g. UUID Components Breakdown

Setiap UUID punya struktur internal yang berbeda per version.

**UUID v4 (Random) Structure:**

```
550e8400-e29b-41d4-a716-446655440000
│      │ │  │ │  │ │  │ │          │
└──┬──┘ │  │ │  │ │  │ └────┬────┘
  Random │  │ │  │ │  │    Random
         │  │ │  │ │  │
    Version (4) │  │ │
              Variant │
                Random
```

**UUID v7 (Timestamp) Structure:**

```
018e1e5e-7c8a-7xxx-xxxx-xxxxxxxxxxxx
│      │ │  │ │  │ │  │ │          │
└──┬──┘ │  │ │  │ │  │ └────┬────┘
Timestamp  │  │ │  │ │    Random
      More │  │ │  │
     Time │  │ │
    Version (7)│
          Variant
            Random

First 48 bits: Millisecond timestamp
Remaining: Random + version bits
```

**Extracting Timestamp dari UUID v7:**

```sql
-- Conceptual (actual implementation varies by extension)
-- First 48 bits = Unix timestamp in milliseconds

-- Example UUID v7:
-- 018e1e5e-7c8a-7xxx-xxxx-xxxxxxxxxxxx
--      ↓
-- 018e1e5e7c8a = timestamp portion
-- Can extract: ~2024-03-15 10:30:45
```

### h. Conversion: String ↔ UUID

PostgreSQL seamlessly convert antara string dan UUID.

```sql
-- String → UUID (implicit cast)
SELECT '550e8400-e29b-41d4-a716-446655440000'::UUID;
-- Works! ✅

-- UUID → String (implicit cast for display)
SELECT uuid_value FROM uuid_example;
-- Displayed as string, but stored as UUID

-- Explicit cast to TEXT
SELECT uuid_value::TEXT FROM uuid_example;
SELECT CAST(uuid_value AS TEXT) FROM uuid_example;

-- Invalid format
SELECT 'not-a-uuid'::UUID;
-- ERROR: invalid input syntax for type uuid

-- Valid formats (all equivalent):
SELECT '550e8400-e29b-41d4-a716-446655440000'::UUID;  -- Standard
SELECT '550e8400e29b41d4a716446655440000'::UUID;      -- No dashes
SELECT '{550e8400-e29b-41d4-a716-446655440000}'::UUID; -- Braces
-- All stored identically as 16 bytes
```

### i. Performance Implications (Preview)

Video menyebutkan primary key discussion akan datang, tapi gives preview.

**UUID v4 sebagai Primary Key: ❌ Problematic**

```sql
CREATE TABLE bad_example (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY
);

-- Masalah:
-- Insert order: Random
-- Index (B-tree): Requires sorting
-- Result: Index fragmentation ❌

-- Insertion pattern:
-- Insert: 9b8c7e6d-... (goes to middle of index)
-- Insert: 1a2b3c4d-... (goes to beginning)
-- Insert: c3d2e1f0-... (goes to end)
-- Index pages: Constantly splitting and rebalancing ❌
```

**UUID v7 sebagai Primary Key: ✅ Good**

```sql
CREATE TABLE good_example (
    id UUID DEFAULT uuid_v7() PRIMARY KEY  -- Assuming extension
);

-- Insertion pattern:
-- Insert: 018e1e5e-... (appends to end of index)
-- Insert: 018e1e5f-... (appends to end)
-- Insert: 018e1e60-... (appends to end)
-- Index pages: Sequential growth ✅
```

**Preview Comparison:**

```
B-tree Index (simplified):

UUID v4 (random):
[1a2b...] [9b8c...] [c3d2...] ← Fragmented, many splits
    ↓         ↓         ↓
  Pages constantly reorganized ❌

UUID v7 (sequential):
[018e1e5e...] [018e1e5f...] [018e1e60...] ← Sequential
                                 ↓
                            Append-only, efficient ✅

BIGINT (serial):
[1] [2] [3] [4] [5] ... ← Most efficient ✅✅
```

## 3. Hubungan Antar Konsep

### Evolution dari Previous Videos:

```
Video: Binary Data (BYTEA)
    └─ MD5 hash storage optimization
         └─ Cast MD5::UUID untuk efficiency
              ↓
Video: UUID sebagai Type
    └─ UUID native type (16 bytes)
         └─ Better than TEXT (40 bytes)
              ↓
Future: Primary Keys & Indexes
    └─ UUID v7 vs BIGINT
         └─ B-tree fragmentation
```

### Decision Tree untuk ID Generation:

```
Butuh generate IDs?

├─ Single database, no distribution
│   └─ BIGINT SERIAL ✅ (simplest, most efficient)
│
├─ Multiple systems, need coordination
│   ├─ Can use central ID service?
│   │   └─ BIGINT from central service ✅
│   └─ Cannot coordinate?
│       └─ UUID v7 ✅ (distributed + sortable)
│
├─ Client-side generation needed
│   └─ UUID v7 ✅ (or UUID v4 for secondary IDs)
│
└─ Public-facing IDs (security by obscurity)
    └─ UUID (any version) ⚠️ (but still need auth!)
```

### Storage Efficiency Hierarchy:

```
Most Efficient
    ↑
BIGINT (8 bytes)             For sequential IDs
    ↑
UUID native (16 bytes)       For distributed IDs
    ↑
TEXT (40 bytes)              ❌ Never for UUIDs!
    ↑
Least Efficient
```

### UUID Version Selection:

```
Need UUID?

├─ For primary key
│   └─ UUID v7 ✅ (sortable, timestamp-based)
│
├─ For secondary identifier
│   └─ UUID v4 ✅ (random, built-in)
│
├─ Deterministic UUID (same input = same UUID)
│   └─ UUID v5 ✅ (SHA-1 based)
│
└─ Legacy compatibility
    └─ UUID v1 (but consider privacy: exposes MAC)
```

## 4. Catatan Tambahan / Insight

### Tips Praktis:

1. **Always Use Native UUID Type:**

   ```sql
   -- ❌ JANGAN: Store sebagai TEXT
   CREATE TABLE items (
       id TEXT  -- 40 bytes, slow comparison
   );

   -- ✅ LAKUKAN: Store sebagai UUID
   CREATE TABLE items (
       id UUID  -- 16 bytes, fast comparison
   );
   ```

2. **Default Value Pattern:**

   ```sql
   -- Pattern: UUID sebagai default
   CREATE TABLE orders (
       id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
       created_at TIMESTAMP DEFAULT NOW()
   );

   -- Insert tanpa ID
   INSERT INTO orders DEFAULT VALUES;
   -- ID auto-generated!
   ```

3. **UUID v7 Extension Check:**

   ```sql
   -- Check if uuid v7 available
   SELECT * FROM pg_available_extensions WHERE name LIKE '%uuid%';

   -- Common extensions:
   -- - uuid-ossp (has v1, v3, v4, v5)
   -- - pg_uuidv7 (community extension for v7)
   -- - Some managed providers include v7 by default
   ```

4. **Validation Pattern:**

   ```sql
   -- Validate UUID format before insert
   CREATE FUNCTION is_valid_uuid(text) RETURNS BOOLEAN AS $$
   BEGIN
       PERFORM $1::UUID;
       RETURN TRUE;
   EXCEPTION WHEN OTHERS THEN
       RETURN FALSE;
   END;
   $$ LANGUAGE plpgsql;

   -- Usage
   SELECT is_valid_uuid('550e8400-e29b-41d4-a716-446655440000');  -- true
   SELECT is_valid_uuid('not-a-uuid');  -- false
   ```

5. **Comparison Performance:**

   ```sql
   -- UUID comparison: Fast (16-byte binary compare)
   SELECT * FROM users WHERE id = '550e8400-...'::UUID;
   -- Uses index efficiently ✅

   -- TEXT comparison: Slower (40-byte string compare + collation)
   SELECT * FROM users WHERE id_text = '550e8400-...';
   -- Slower even with index ⚠️
   ```

### Kesalahan Umum:

```sql
-- ❌ KESALAHAN 1: Store UUID sebagai TEXT
CREATE TABLE users (
    id TEXT PRIMARY KEY
);
-- Waste 24 bytes per row + slower queries

-- ✅ FIX:
CREATE TABLE users (
    id UUID PRIMARY KEY
);


-- ❌ KESALAHAN 2: Gunakan UUID v4 sebagai primary key
CREATE TABLE orders (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY
);
-- Index fragmentation! (will learn more later)

-- ✅ FIX: Gunakan UUID v7 atau BIGINT
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
    -- Or: id UUID DEFAULT uuid_v7() PRIMARY KEY
);


-- ❌ KESALAHAN 3: Manual UUID formatting
INSERT INTO items VALUES (
    CONCAT(
        LPAD(FLOOR(RANDOM() * 1000000)::TEXT, 8, '0'),
        '-', ...
    )
);
-- Complex, error-prone, tidak standard

-- ✅ FIX: Gunakan gen_random_uuid()
INSERT INTO items VALUES (gen_random_uuid());


-- ❌ KESALAHAN 4: Mix UUID versions tanpa reason
CREATE TABLE data (
    id UUID DEFAULT gen_random_uuid(),  -- v4
    correlation_id UUID  -- Dari external system, unknown version
);
-- Bisa confusing, tapi kadang unavoidable

-- ✅ BETTER: Document version usage
COMMENT ON COLUMN data.id IS 'UUID v4 - internal generated';
COMMENT ON COLUMN data.correlation_id IS 'UUID any version - from external';


-- ❌ KESALAHAN 5: Assume UUID = secure/unpredictable
-- URLs: /api/items/018e1e5e-7c8a-...
-- UUID v7 has timestamp → bisa predicted/enumerated!
-- Tetap butuh proper authentication ✅

-- Security catatan:
-- UUIDs are identifiers, NOT security tokens
-- Always implement proper auth/authz
```

### Advanced Patterns:

**Pattern 1: Composite IDs**

```sql
-- Combine BIGINT (internal) dengan UUID (external)
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- Internal, efficient
    public_id UUID DEFAULT gen_random_uuid() UNIQUE,     -- External, non-guessable
    customer_id BIGINT,
    total NUMERIC
);

CREATE INDEX ON orders(public_id);

-- Internal queries: Use id (efficient)
SELECT * FROM orders WHERE id = 12345;

-- External API: Use public_id (secure)
GET /api/orders/550e8400-...
```

**Pattern 2: UUID Namespace (v5)**

```sql
-- Create deterministic UUIDs
-- Same input = same UUID (useful untuk idempotency)

-- Example: Generate UUID dari email
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

SELECT uuid_generate_v5(
    uuid_ns_url(),  -- Namespace
    'user@example.com'  -- Name
);
-- Always returns: same UUID untuk same email ✅

-- Use case: Deduplication
INSERT INTO users (id, email)
VALUES (
    uuid_generate_v5(uuid_ns_url(), 'user@example.com'),
    'user@example.com'
)
ON CONFLICT (id) DO NOTHING;
-- Automatically prevents duplicates!
```

**Pattern 3: Migration dari STRING ke UUID**

```sql
-- Existing table dengan TEXT IDs
CREATE TABLE legacy_table (
    id TEXT PRIMARY KEY
);

-- Step 1: Add UUID column
ALTER TABLE legacy_table ADD COLUMN id_uuid UUID;

-- Step 2: Convert existing data
UPDATE legacy_table SET id_uuid = id::UUID;

-- Step 3: Swap primary key (requires downtime)
ALTER TABLE legacy_table DROP CONSTRAINT legacy_table_pkey;
ALTER TABLE legacy_table ALTER COLUMN id DROP NOT NULL;
ALTER TABLE legacy_table ALTER COLUMN id_uuid SET NOT NULL;
ALTER TABLE legacy_table ADD PRIMARY KEY (id_uuid);
ALTER TABLE legacy_table DROP COLUMN id;
ALTER TABLE legacy_table RENAME COLUMN id_uuid TO id;
```

**Pattern 4: UUID untuk Soft Deletes**

```sql
CREATE TABLE documents (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    title TEXT,
    deleted_at TIMESTAMP,
    deleted_id UUID  -- UUID from deletion event
);

-- Mark as deleted dengan unique ID
UPDATE documents
SET deleted_at = NOW(),
    deleted_id = gen_random_uuid()
WHERE id = '...';

-- Track deletion events
CREATE TABLE deletion_events (
    id UUID PRIMARY KEY,
    table_name TEXT,
    record_id UUID,
    deleted_at TIMESTAMP DEFAULT NOW()
);

-- Correlate deletions across tables
```

### Real-World Scenarios:

**Scenario 1: Microservices**

```sql
-- Order Service
CREATE TABLE orders (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    customer_id UUID NOT NULL
);

-- Payment Service
CREATE TABLE payments (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    order_id UUID NOT NULL  -- References order from different service
);

-- Keuntungan:
-- - No cross-service DB calls untuk generate IDs
-- - Each service independent
-- - Easy to merge data untuk analytics
```

**Scenario 2: Offline-First Mobile App**

```javascript
// Mobile app generates UUID offline
const orderId = crypto.randomUUID();

// Store locally
localDB.insert({ id: orderId, items: [...] });

// Sync when online (no ID conflict!)
api.post('/orders', { id: orderId, items: [...] });

// PostgreSQL
INSERT INTO orders (id, items)
VALUES ('550e8400-...', ...);
-- No ID collision dengan orders dari other users/devices ✅
```

**Scenario 3: Multi-Tenant dengan UUID**

```sql
-- Tenant-scoped UUIDs
CREATE TABLE tenants (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    name TEXT
);

CREATE TABLE users (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    tenant_id UUID REFERENCES tenants(id),
    email TEXT,
    UNIQUE(tenant_id, email)  -- Email unique per tenant
);

-- Partition by tenant_id untuk performance
CREATE TABLE users_partitioned (
    id UUID,
    tenant_id UUID,
    email TEXT
) PARTITION BY LIST (tenant_id);
```

### Analogi:

**UUID seperti:**

- **BIGINT** = Nomor antrian (sequential, predictable, efficient)
- **UUID v4** = Nomor lotre acak (unpredictable, tapi random order)
- **UUID v7** = Nomor tiket dengan timestamp (sorted by time, tapi unique)

**Storage Comparison:**

- **TEXT UUID** = Menulis UUID dengan pen di kertas (40 characters)
- **UUID Type** = Barcode sticker (16 bytes binary, compact)

**Distribution:**

- **Central ID server** = Ambil nomor antrian dari satu loket
- **UUID** = Setiap orang bikin nomor sendiri, guaranteed unique

## 5. Kesimpulan

UUID adalah tipe data 128-bit (16 bytes) di PostgreSQL yang sangat berguna untuk generating unique identifiers tanpa koordinasi antar sistem. PostgreSQL menyediakan native UUID type yang jauh lebih efficient (16 bytes) dibanding menyimpan sebagai TEXT (40 bytes).

**Key Points:**

1. **UUID Type**: Native type, 16 bytes storage, efficient comparison
2. **Universally Unique**: Collision probability effectively zero
3. **Built-in Generation**: `gen_random_uuid()` untuk UUID v4
4. **UUID v7**: Timestamp-based, sortable, better untuk primary keys (butuh extension)
5. **UUID v4**: Random, NOT recommended untuk primary keys

**Storage Comparison:**

```
TEXT:   40 bytes  (36 chars + 4 overhead)  ❌
UUID:   16 bytes  (native binary)          ✅
Savings: 60% reduction untuk large tables
```

**Best Practices:**

- ✅ Always store UUIDs as UUID type, never TEXT
- ✅ Use `gen_random_uuid()` untuk automatic generation
- ✅ Use UUID v7 untuk primary keys (when available)
- ❌ Avoid UUID v4 untuk primary keys (index fragmentation)
- ✅ Document which UUID version you're using

**Use Cases:**

- Distributed systems (microservices)
- Client-side ID generation
- Multi-database synchronization
- Public-facing identifiers
- Offline-first applications

**Looking Ahead:**
Video ini adalah foundation untuk discussion yang akan datang tentang:

- Primary keys (BIGINT vs UUID)
- B-tree indexes
- Index fragmentation
- UUID v7 vs sequential IDs

**Key takeaway:** UUID adalah tool yang powerful untuk distributed ID generation. Store sebagai native UUID type untuk maximum efficiency (16 bytes vs 40 bytes untuk TEXT). Untuk primary keys, prefer UUID v7 (sortable) atau stick dengan BIGINT SERIAL (most efficient). UUID v4 bagus untuk secondary identifiers, tapi avoid sebagai primary key karena index fragmentation issues yang akan dijelaskan di video mendatang.
