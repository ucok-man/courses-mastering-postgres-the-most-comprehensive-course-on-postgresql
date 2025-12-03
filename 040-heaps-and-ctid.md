# 📝 Heaps dan CTIDs - Storage Internal PostgreSQL

## 1. Ringkasan Singkat

Video ini menjelaskan mekanisme storage internal PostgreSQL untuk menjawab pertanyaan: "Bagaimana index tahu cara kembali ke table untuk mengambil full row data?" Materi mencakup konsep heap storage (cara PostgreSQL menyimpan data di disk), pages, CTIDs (Tuple IDs), dan bagaimana index menggunakan CTID sebagai pointer untuk kembali ke table. Ini adalah deep dive ke level yang lebih rendah untuk membangun pemahaman fundamental tentang cara kerja indexes.

## 2. Konsep Utama

### a. Review: Pointer dari Index ke Table

**Recall dari video sebelumnya:**

```
Query: SELECT * FROM users WHERE last_name = 'Smith';

Step 1: Traverse index
        idx_last_name → Find 'Smith'

Step 2: Index hanya punya last_name column
        Query butuh "*" (semua kolom)

Step 3: Gunakan POINTER untuk kembali ke table
        ↓
        Pertanyaan: Apa itu pointer? Bagaimana cara kerjanya?
```

**Masalah yang harus dipecahkan:**

- Index adalah struktur data terpisah dari table
- Setelah traverse index, perlu cara untuk "jump" langsung ke row yang tepat di table
- Tanpa pointer: Harus scan seluruh table lagi (lambat!)
- Dengan pointer: Langsung ke lokasi exact row (cepat!)

**Pertanyaan kunci video ini:**

> "What is that connection between the separate data structure and the rest of the data that we likely need?"

### b. Heap Storage - Cara PostgreSQL Menyimpan Data

**Definisi Heap:**

- **Heap** adalah struktur penyimpanan data di PostgreSQL
- Nama yang tepat karena seperti "pile" atau "tumpukan"
- Data ditulis ke lokasi mana pun yang ada space tersedia

**Karakteristik heap:**

```
Heap = "Just put the rows wherever there is space"

Tidak ada organizing principle khusus:
- Bisa di page 0
- Bisa di page 60
- Bisa di page 432
- Yang penting: Ada space kosong
```

**Mengapa menggunakan heap?**

| Aspek                  | Benefit                                           |
| ---------------------- | ------------------------------------------------- |
| **INSERT performance** | Sangat cepat! Tidak perlu rearrange existing data |
| **Write efficiency**   | Cukup cari blank space dan tulis di sana          |
| **Simplicity**         | Tidak perlu maintain sorted order                 |

**Visualisasi heap storage:**

```
┌─────────────────────────────────────────────┐
│           PostgreSQL Database               │
├─────────────────────────────────────────────┤
│  Page 0  │  Page 1  │  Page 2  │  Page 3   │
├──────────┼──────────┼──────────┼───────────┤
│ Row 1    │ Row 7    │          │ Row 15    │
│ Row 2    │ Row 8    │ (empty)  │ Row 16    │
│ Row 3    │          │          │           │
│          │ (empty)  │          │ (empty)   │
└──────────┴──────────┴──────────┴───────────┘
    ↑
Rows ditempatkan di mana pun ada space,
tidak ada sorted order!
```

**Contoh proses INSERT:**

```
INSERT INTO users VALUES ('Alice', 'Smith', ...);
        ↓
PostgreSQL: "Di mana ada space kosong?"
        ↓
Scan pages: Page 0 full, Page 1 has space!
        ↓
Write to: Page 1, position 9
        ↓
Done! Very fast.
```

### c. Pages - Unit Storage di PostgreSQL

**Apa itu Page?**

- **Page** = Unit penyimpanan dasar di PostgreSQL
- Equal-sized blocks of data
- Default size: 8KB per page
- Semua data (tables, indexes) disimpan dalam pages

**Struktur pages:**

```
Database = Collection of pages

┌──────────────────────┐
│      Page 0          │  8KB
├──────────────────────┤
│      Page 1          │  8KB
├──────────────────────┤
│      Page 2          │  8KB
├──────────────────────┤
│      Page 3          │  8KB
├──────────────────────┤
│      ...             │
├──────────────────────┤
│      Page 999        │  8KB
└──────────────────────┘
```

**Inside a single page:**

```
┌────────────────────────────────────┐
│         Page 5 (8KB)               │
├────────────────────────────────────┤
│ Position 0: Row data (120 bytes)  │
│ Position 1: Row data (200 bytes)  │
│ Position 2: Row data (150 bytes)  │
│ Position 3: Row data (180 bytes)  │
│ ...                                │
│ Position 15: Row data (95 bytes)  │
│ (remaining space available)        │
└────────────────────────────────────┘
```

**Key points:**

- Pages di-number mulai dari 0: Page 0, Page 1, Page 2, ...
- Di dalam page, rows punya positions (offsets)
- Position juga di-number dari 0

### d. CTID (Tuple ID) - Physical Row Identifier

**Definisi CTID:**

- **CTID** = Physical location identifier untuk sebuah row
- Format: `(page_number, position_within_page)`
- Contoh: `(0,1)` = Page 0, Position 1

**CTID adalah hidden system column:**

```sql
-- SELECT * TIDAK include CTID
SELECT * FROM reservations;
-- id | room_id | reservation_period | booking_status
-- ---|---------|--------------------|--------------
-- 1  | 101     | [...]              | confirmed

-- Tapi CTID tetap ada, harus explicitly select
SELECT ctid, * FROM reservations;
-- ctid  | id | room_id | reservation_period | booking_status
-- ------|----|---------|--------------------|---------------
-- (0,1) | 1  | 101     | [...]              | confirmed
-- (0,2) | 2  | 102     | [...]              | confirmed
-- (0,3) | 3  | 101     | [...]              | canceled
```

**Penjelasan hasil:**

```
(0,1) → Page 0, Position 1 → Entire row #1 exists here
(0,2) → Page 0, Position 2 → Entire row #2 exists here
(0,3) → Page 0, Position 3 → Entire row #3 exists here
```

**CTID memberikan physical disk location:**

```
Database di disk:

Page 0:
├─ Position 1: [1, 101, '[2024-09-01...]', 'confirmed']  ← Row dengan ctid (0,1)
├─ Position 2: [2, 102, '[2024-09-02...]', 'confirmed']  ← Row dengan ctid (0,2)
└─ Position 3: [3, 101, '[2024-09-03...]', 'canceled']   ← Row dengan ctid (0,3)

Dengan CTID, bisa LANGSUNG jump ke physical location!
```

### e. Querying by CTID (dan Mengapa TIDAK Boleh Dilakukan)

**Teknis bisa query by CTID:**

```sql
-- Ambil row di Page 0, Position 2
SELECT * FROM reservations WHERE ctid = '(0,2)';
-- ✅ Technically works

-- Result:
-- id | room_id | reservation_period | booking_status
-- ---|---------|--------------------|--------------
-- 2  | 102     | [...]              | confirmed
```

**⚠️ TAPI JANGAN LAKUKAN INI! Mengapa?**

**Alasan 1: CTIDs are VOLATILE (berubah-ubah)**

```sql
-- Original location
SELECT ctid, * FROM users WHERE id = 1;
-- ctid  | id | name
-- ------|----|---------
-- (0,2) | 1  | 'Alice'

-- UPDATE row (data jadi lebih besar)
UPDATE users SET bio = 'Very long text...' WHERE id = 1;

-- CTID berubah!
SELECT ctid, * FROM users WHERE id = 1;
-- ctid   | id | name
-- -------|----|---------
-- (84,5) | 1  | 'Alice'  ← Pindah ke Page 84!
```

**Mengapa CTID berubah?**

```
Scenario:
1. Row originally di Page 0, Position 2
2. UPDATE membuat row lebih besar
3. Page 0 tidak punya space cukup
4. PostgreSQL move row ke Page 84 yang punya space
5. CTID berubah dari (0,2) ke (84,5)
```

**Alasan 2: VACUUM merearrange rows**

```sql
-- Before VACUUM
SELECT ctid, id FROM products;
-- ctid  | id
-- ------|----
-- (0,1) | 1
-- (0,2) | 2  ← Deleted
-- (0,3) | 3
-- (5,8) | 4

DELETE FROM products WHERE id = 2;

-- After VACUUM (cleanup dead tuples + compact space)
SELECT ctid, id FROM products;
-- ctid  | id
-- ------|----
-- (0,1) | 1
-- (0,2) | 3  ← CTID berubah! Tadinya (0,3)
-- (0,3) | 4  ← CTID berubah! Tadinya (5,8)
```

**Summary: JANGAN gunakan CTID sebagai identifier!**

| Karakteristik              | CTID                          | Primary Key      |
| -------------------------- | ----------------------------- | ---------------- |
| **Stable**                 | ❌ Berubah saat UPDATE/VACUUM | ✅ Tidak berubah |
| **Deterministic**          | ❌ Unpredictable              | ✅ Deterministic |
| **Volatile**               | ✅ Ya                         | ❌ Tidak         |
| **Suitable as identifier** | ❌ TIDAK!                     | ✅ Ya            |

**Instruktur's warning:**

> "Don't do that. These CTIDs can and will change. They're not primary keys, they're not stable, they're not deterministic, they're volatile."

### f. CTID sebagai Pointer dalam Index - The Missing Link!

**Reveal: Apa itu "pointer" dalam index?**

```
Pertanyaan dari video sebelumnya:
"Every index contains a pointer back to the table"

Jawaban sekarang:
"Every index contains the CTID"
```

**CTID = The actual pointer!**

```
Index Entry:
┌─────────────────────────────┐
│ Indexed value | CTID        │
├───────────────┼─────────────┤
│ 'Smith'       | (0,10)      │  ← CTID adalah pointer!
│ 'Johnson'     | (2,5)       │
│ 'Williams'    | (5,8)       │
└───────────────┴─────────────┘
```

**Complete picture: Bagaimana index lookup bekerja**

```sql
SELECT * FROM users WHERE last_name = 'Smith';
```

**Step-by-step dengan CTID:**

```
Step 1: Traverse index idx_last_name
        ↓
        Binary search atau tree traversal
        ↓
        Found: 'Smith' → CTID: (0,10)

Step 2: Use CTID as pointer
        ↓
        "Go to Page 0, Position 10"
        ↓
        Direct jump! No scanning needed.

Step 3: Retrieve full row from table
        ↓
        Page 0, Position 10:
        [id: 123, first: 'John', last: 'Smith', email: 'john@...']
        ↓
        Return to user
```

**Visualisasi complete flow:**

```
┌──────────────────┐         ┌─────────────────────────┐
│  Index           │         │  Table (Heap)           │
│  (idx_last_name) │         │                         │
├──────────────────┤         │  Page 0:                │
│ 'Johnson' (2,5)  │         │  ├─ Pos 10: [Smith...] │◄─┐
│ 'Smith'   (0,10) │─────────┼──┘                      │  │
│ 'Williams'(5,8)  │         │  Page 2:                │  │
└──────────────────┘         │  ├─ Pos 5: [Johnson...] │  │
        ↓                    │                         │  │
   1. Find 'Smith'           │  Page 5:                │  │
   2. Get CTID: (0,10)       │  └─ Pos 8: [Williams..]│  │
   3. Jump to table ─────────────────────────────────────┘
```

**Mengapa ini sangat efisien?**

```
Without CTID (theoretical):
Query → Index → "I found 'Smith' somewhere"
     → Must scan entire table to find it
     → O(n) complexity for table scan

With CTID (actual):
Query → Index → "I found 'Smith' at (0,10)"
     → Direct jump to Page 0, Position 10
     → O(1) complexity for row retrieval
```

### g. Key Insight: Index Storage vs Table Storage

**Index stores:**

```
1. Indexed column value(s)
2. CTID (pointer to table)

Example for idx_last_name:
┌──────────┬────────┐
│ last_name│ CTID   │
├──────────┼────────┤
│ 'Doe'    │ (1,3)  │
│ 'Smith'  │ (0,10) │
└──────────┴────────┘
```

**Table stores:**

```
Complete row data at physical location

Page 0, Position 10:
[id: 123, first_name: 'John', last_name: 'Smith',
 email: 'john@example.com', created_at: '2024-01-01', ...]
```

**Division of labor:**

| Component | Responsibility               | Optimization                        |
| --------- | ---------------------------- | ----------------------------------- |
| **Index** | Fast lookup by indexed value | B-tree, sorted, optimized traversal |
| **CTID**  | Bridge between index & table | Direct physical address             |
| **Heap**  | Store complete row data      | Fast writes, flexible storage       |

### h. Implications untuk Index Design

**Sekarang kita tahu:**

1. **Index HARUS store CTID**

   ```
   Index size = (indexed data) + (CTID per entry)

   CTID size: 6 bytes (4 bytes page + 2 bytes position)
   ```

2. **Index overhead**

   ```
   Every index entry = indexed value + 6 bytes CTID

   Example:
   - Index on INTEGER (4 bytes) → 4 + 6 = 10 bytes per entry
   - Index on TEXT (avg 20 bytes) → 20 + 6 = 26 bytes per entry
   ```

3. **Covering indexes (hint untuk nanti)**

   ```
   Normal index: value + CTID
   Covering index: value + additional columns + CTID

   Benefit: Bisa return data dari index saja tanpa table lookup!
   (Will be covered in later videos)
   ```

## 3. Hubungan Antar Konsep

### Alur Pemahaman Complete Picture:

```
1. User Query
   ↓
   SELECT * FROM users WHERE last_name = 'Smith'

2. Query Planner
   ↓
   "Use index idx_last_name"

3. Index Traversal (B-tree, covered later)
   ↓
   Navigate tree structure
   ↓
   Find: 'Smith' → Associated CTID: (0,10)

4. CTID Lookup
   ↓
   "Go to Heap: Page 0, Position 10"

5. Heap Access
   ↓
   Read Page 0 from disk (or cache)
   ↓
   Get Position 10 within that page
   ↓
   Full row data retrieved

6. Return Result
   ↓
   [id: 123, first_name: 'John', last_name: 'Smith', ...]
```

### Hierarchy of Storage Concepts:

```
PostgreSQL Database
│
├─ Heap Storage (Table data)
│  ├─ Pages (8KB blocks)
│  │  ├─ Page 0
│  │  │  ├─ Position 0 → Row
│  │  │  ├─ Position 1 → Row
│  │  │  └─ Position n → Row
│  │  ├─ Page 1
│  │  │  └─ ...
│  │  └─ Page n
│  │
│  └─ CTID system column
│     └─ (page_number, position) → Physical location
│
└─ Index Storage (Separate structure)
   ├─ B-tree (or other structure)
   ├─ Indexed values (sorted/organized)
   └─ CTIDs (pointers to heap)
```

### Mengapa Heap + CTID Design Masuk Akal:

```
Design Goals:
├─ Fast writes (inserts)
│  └─ Solution: Heap (no sorting, write anywhere)
│
├─ Fast reads (selects with WHERE clause)
│  └─ Solution: Index (organized structure)
│
└─ Fast connection between index & table
   └─ Solution: CTID (direct physical address)

Result: Optimal balance for OLTP workloads
```

### Trade-offs dari Design Ini:

**Pros:**

```
✅ INSERT performance excellent (heap = write anywhere)
✅ Index lookups very fast (B-tree + direct CTID access)
✅ Flexible schema (easy to add/remove indexes)
```

**Cons:**

```
❌ Full table scans slow (no inherent order in heap)
   → Solution: Use indexes!

❌ Index maintenance cost on writes (must update CTIDs)
   → Solution: Only index what you need

❌ VACUUM needed to reclaim space (deleted rows leave gaps)
   → Solution: Regular maintenance (autovacuum)
```

### Mental Model: Analogi Library

```
HEAP = Gudang buku (storage room)
       ├─ Shelves = Pages
       ├─ Positions on shelf = Row positions
       └─ Books arranged by acquisition order (not sorted)

INDEX = Catalog system
        ├─ Organized alphabetically/by topic
        ├─ Entry: "Book Title" → "Shelf 5, Position 3"
        └─ CTID = "Shelf 5, Position 3" (location code)

Lookup process:
1. Check catalog (index) → "Pride and Prejudice"
2. Get location: Shelf 5, Position 3 (CTID)
3. Walk to Shelf 5 (page)
4. Grab book at Position 3 (row)
5. Done! No need to search entire warehouse.
```

### Progression of Understanding:

```
Video 1 (Introduction to Indexes):
"Index contains a pointer to table"
        ↓
        What is this pointer? 🤔

Video 2 (This video):
"Pointer = CTID = (page, position)"
        ↓
        Aha! Now I understand the mechanism!

Next videos:
"How does index organize data for fast lookup?"
        ↓
        B-tree structure, etc.
```

## 4. Kesimpulan

**Key Takeaways:**

1. **Heap Storage:**

   - PostgreSQL menggunakan heap storage untuk table data
   - Heap = "pile" - rows ditempatkan di mana pun ada space
   - Benefit: INSERT sangat cepat (tidak perlu maintain order)

2. **Pages:**

   - Unit storage dasar di PostgreSQL
   - Default size: 8KB per page
   - Numbered mulai dari 0: Page 0, Page 1, Page 2, ...
   - Di dalam page ada positions untuk rows

3. **CTID (Tuple ID):**

   - Physical location identifier: `(page_number, position)`
   - Hidden system column (tidak terlihat di SELECT \*)
   - Format: `(0,10)` = Page 0, Position 10

4. **⚠️ JANGAN gunakan CTID sebagai identifier:**

   - ❌ Volatile: Berubah saat UPDATE
   - ❌ Not stable: Berubah saat VACUUM
   - ❌ Not deterministic: Unpredictable
   - ✅ Gunakan PRIMARY KEY untuk identifikasi!

5. **CTID adalah "Pointer" dalam Index:**

   ```
   Index Entry = (indexed_value, CTID)

   Example:
   'Smith' → (0,10)

   Meaning: "Value 'Smith' is located at Page 0, Position 10"
   ```

6. **Complete Lookup Flow:**

   ```
   Query → Index (find value, get CTID)
        → Heap (jump to CTID location)
        → Return full row
   ```

7. **Efficiency:**
   - Tanpa CTID: Must scan table (slow)
   - Dengan CTID: Direct jump O(1) (fast!)

**Prinsip emas:**

> "Every index contains the CTID, because that is the actual pointer that gets you back to the table to find the rest of your data"

**Yang sudah kita pahami sekarang:**

- ✅ Index adalah separate structure (Video 1)
- ✅ Index contains copy of data (Video 1)
- ✅ Index contains pointer = **CTID** (Video 2) ← New!
- ✅ Heap storage mechanism (Video 2) ← New!

**Yang akan dipelajari selanjutnya:**

- Bagaimana index organize data untuk fast traversal? (B-tree structure)
- Index types dan kapan menggunakan apa
- Practical index creation strategies
- Performance tuning dengan indexes

Dengan memahami heap, pages, dan CTIDs, kita sekarang punya complete picture tentang bagaimana index benar-benar bekerja di level internal PostgreSQL! 🎯
