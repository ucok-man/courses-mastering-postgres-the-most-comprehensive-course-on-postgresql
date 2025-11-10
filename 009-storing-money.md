# 📝 Menyimpan Data Uang (Money) di PostgreSQL

## 1. Ringkasan Singkat

Video ini membahas cara yang **benar dan salah** untuk menyimpan data uang (money) di PostgreSQL. Meskipun PostgreSQL memiliki tipe data `MONEY`, video ini **sangat menyarankan untuk TIDAK menggunakannya** karena berbagai masalah serius. Sebagai gantinya, direkomendasikan dua pendekatan: menyimpan sebagai **INTEGER** (dalam unit terkecil seperti cents) atau sebagai **NUMERIC** dengan precision dan scale yang tepat.

## 2. Konsep Utama

### a. Tipe Data MONEY: Kenapa TIDAK Direkomendasikan

PostgreSQL memiliki tipe data `MONEY`, tetapi **sangat tidak disarankan** untuk production use.

**Deklarasi dan penggunaan:**

```sql
-- Cara deklarasi
CREATE TABLE money_example (
    item_name TEXT,
    price MONEY
);

-- Insert berbagai format
INSERT INTO money_example (item_name, price) VALUES
    ('Laptop', 999.99),           -- Float literal
    ('Mouse', 25.50),              -- Decimal literal
    ('Keyboard', 75),              -- Integer literal
    ('Headphones', '$129.99');     -- Currency-formatted string

-- Semua format di atas VALID
SELECT * FROM money_example;
-- Result:
-- Laptop      | $999.99
-- Mouse       | $25.50
-- Keyboard    | $75.00
-- Headphones  | $129.99
```

**Flexibility input:**

- ✅ Accept float literals
- ✅ Accept integer literals
- ✅ Accept currency-formatted strings (`'$129.99'`)
- ✅ Automatically format dengan currency symbol

**Terlihat bagus? WAIT! Ada masalah besar...**

### b. Masalah #1: Loss of Precision

**Problem:**
Tipe `MONEY` **hanya mendukung 2 decimal places**.

**Demonstrasi:**

```sql
-- Test precision loss
SELECT 1.99::MONEY;
-- Result: $1.99 ✅

SELECT 99.99::MONEY;
-- Result: $99.99 ✅

SELECT 99.876::MONEY;
-- Result: $99.88 ⚠️ (rounded, kehilangan digit ke-3!)
```

**Implikasi:**

**✅ OK untuk USD standard (2 decimal places):**

```sql
-- US Dollars dengan cents standar
SELECT 100.78::MONEY;
-- Result: $100.78 (perfect)
```

**❌ MASALAH untuk fractional cents:**

```sql
-- Jika butuh fractional cents (misal: tax calculations)
SELECT 100.789::MONEY;
-- Result: $100.79 (kehilangan precision!)

-- Financial calculations yang precise
SELECT (100.33 / 3)::MONEY;
-- Result: $33.44 (harusnya $33.443333...)
```

**❌ MASALAH BESAR untuk cryptocurrency:**

```sql
-- Bitcoin atau crypto dengan banyak decimal places
SELECT 0.00012345::MONEY;
-- Result: $0.00 (DISASTER! Kehilangan hampir semua value!)

-- Ethereum (18 decimal places) - completely unusable
```

**Kesimpulan:**

> Jika aplikasi Anda **hanya** berurusan dengan US Dollars atau currency dengan 2 decimal places standard, dan **tidak pernah** butuh fractional cents, MUNGKIN `MONEY` OK. **Tapi kenapa ambil risk?**

### c. Masalah #2: Locale Dependency (LC_MONETARY)

**Problem yang SANGAT SERIUS:**
Currency symbol dan formatting bergantung pada `lc_monetary` setting, yang bisa **berubah** dan menyebabkan **data terlihat salah**.

**Demonstrasi masalah:**

**Initial setup (US Dollar):**

```sql
-- Check current locale
SHOW lc_monetary;
-- Result: en_US.UTF-8

-- Display data
SELECT * FROM money_example;
-- Result:
-- Laptop      | $999.99  (US Dollar symbol)
-- Mouse       | $25.50
-- Keyboard    | $75.00
```

**Setelah locale berubah:**

```sql
-- Change locale to Great British Pound
SET lc_monetary = 'en_GB.UTF-8';

-- Display data yang SAMA
SELECT * FROM money_example;
-- Result:
-- Laptop      | £999.99  (Sekarang Pound symbol!)
-- Mouse       | £25.50
-- Keyboard    | £75.00
```

**MASALAH BESAR:**

```
Data SEBENARNYA: $999.99 USD (sekitar £790 GBP)
Data TERLIHAT:   £999.99 GBP (terlihat lebih mahal!)

❌ TIDAK ADA currency conversion yang terjadi!
❌ Hanya SYMBOL yang berubah!
❌ Representasi data menjadi TOTALLY WRONG!
```

**Ilustrasi visual:**

```
Stored value (internal): 999.99

┌─────────────────┬──────────────────┐
│ lc_monetary     │ Display          │
├─────────────────┼──────────────────┤
│ en_US.UTF-8     │ $999.99          │
│ en_GB.UTF-8     │ £999.99          │
│ de_DE.UTF-8     │ 999,99 €         │
│ ja_JP.UTF-8     │ ¥999.99          │
└─────────────────┴──────────────────┘

Value TIDAK BERUBAH, hanya PRESENTASI!
Super confusing dan misleading!
```

**Skenario disaster:**

```sql
-- Scenario: Multi-tenant application
-- Tenant A: US company (stores in USD)
-- Tenant B: UK company (changes lc_monetary)

-- Tenant A stores data
SET lc_monetary = 'en_US.UTF-8';
INSERT INTO invoices VALUES ('Invoice-001', 1000.00::MONEY);
-- Internal: 1000.00, Display: $1000.00

-- Later, Tenant B (or system restart with different locale)
SET lc_monetary = 'en_GB.UTF-8';
SELECT * FROM invoices WHERE invoice_id = 'Invoice-001';
-- Display: £1000.00 (WRONG! Ini seharusnya $1000.00)

-- Accounting nightmare! Values terlihat salah!
```

**Kesimpulan:**

> `MONEY` type **TIDAK MENYIMPAN currency information**. Hanya nilai numerik + formatting bergantung locale. **Sangat berbahaya** untuk aplikasi multi-currency atau sistem yang bisa berubah locale.

### d. Summary: Masalah dengan Tipe MONEY

**Recap dua masalah utama:**

**Problem #1: Limited Precision**

- ✅ **Pro:** Operations dijamin precise
- ❌ **Con:** Hanya 2 decimal places (fractional cents impossible)

**Problem #2: Locale-Dependent Display**

- ❌ **Con:** Currency symbol berubah sesuai `lc_monetary`
- ❌ **Con:** Tidak ada actual currency conversion
- ❌ **Con:** Data bisa misleading/confusing

**Video quote:**

> "I'm going to beg you to not use it to store money in the database."

### e. SOLUSI #1: Store sebagai INTEGER (Recommended!)

**Konsep:**
Simpan uang dalam **unit terkecil** currency (seperti cents untuk USD) sebagai INTEGER.

**Pendekatan:**

```sql
-- Store everything in CENTS (smallest unit)
CREATE TABLE products (
    product_name TEXT,
    price_cents INTEGER  -- Store cents, bukan dollars
);

-- $100.78 = 10078 cents
INSERT INTO products VALUES ('Laptop', 10078);

-- $25.50 = 2550 cents
INSERT INTO products VALUES ('Mouse', 2550);

-- Query
SELECT
    product_name,
    price_cents,
    price_cents / 100.0 AS price_dollars  -- Convert untuk display
FROM products;

-- Result:
-- Laptop  | 10078 | 100.78
-- Mouse   | 2550  | 25.50
```

**Conversion logic:**

```sql
-- Dollars to Cents (storage)
SELECT (100.78 * 100)::INT4;  -- 10078 cents

-- Cents to Dollars (display)
SELECT 10078 / 100.0;  -- 100.78 dollars
```

**Untuk fractional cents (jika dibutuhkan):**

```sql
-- Store in thousandths of a cent (0.001¢)
CREATE TABLE precise_prices (
    product_name TEXT,
    price_millis INTEGER  -- Thousandths of a cent
);

-- $100.789 = 100789 millis (100.789 * 1000)
INSERT INTO precise_prices VALUES ('Item', 100789);

-- Convert back
SELECT 100789 / 1000.0;  -- 100.789
```

**Keuntungan INTEGER approach:**

**1. Performance - Sangat Cepat:**

```
INTEGER operations adalah yang TERCEPAT:
- Addition: native CPU operation
- Comparison: instant
- Indexing: optimal
```

**2. Accuracy - Perfectly Precise:**

```sql
-- No floating-point errors
SELECT (10078 + 2550);  -- 12628 (exact!)

-- Dibanding float
SELECT (100.78 + 25.50)::FLOAT8;  -- 126.28000000000002 (ugh!)
```

**3. Simplicity - Easy to Reason:**

```
Price in cents: 10078
Always an integer, no ambiguity
Can't accidentally introduce precision errors
```

**4. API-Friendly:**

```
Stripe API menggunakan metode ini!
Amount dalam smallest currency unit:
- USD: cents
- JPY: yen (already no decimals)
- EUR: euro cents
```

**Real-world example (Stripe-style):**

```sql
-- Orders table (Stripe-inspired)
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    amount_cents INTEGER,     -- USD cents
    currency CHAR(3),         -- 'USD', 'EUR', 'GBP'
    status TEXT
);

-- Create order for $49.99
INSERT INTO orders VALUES
    (1, 100, 4999, 'USD', 'paid');

-- Query untuk display
SELECT
    order_id,
    amount_cents / 100.0 AS amount_dollars,
    currency
FROM orders;
-- Result: 1 | 49.99 | USD
```

**Trade-offs:**

**✅ Pros:**

- ⚡ Blazing fast (native integer operations)
- ✅ Perfectly accurate (no precision loss)
- 🎯 Simple to understand
- 🔒 Type-safe (can't accidentally use float)
- 📦 Easy to serialize/deserialize untuk APIs

**⚠️ Cons:**

- 🔄 Conversion overhead (multiply on insert, divide on display)
- 🧠 Mental overhead (harus ingat unit: "10078 = $100.78")
- 📝 Application logic harus handle conversion
- ⚠️ Developer bisa lupa convert (show "10078" ke user!)

**Best practices:**

```sql
-- ✅ GOOD: Clear naming
CREATE TABLE invoices (
    amount_cents INTEGER,  -- Jelas: dalam cents
    currency_code CHAR(3)
);

-- ✅ GOOD: Helper functions
CREATE FUNCTION cents_to_dollars(cents INTEGER)
RETURNS NUMERIC(10,2) AS $$
BEGIN
    RETURN cents / 100.0;
END;
$$ LANGUAGE plpgsql;

-- ✅ GOOD: Views untuk display
CREATE VIEW invoices_display AS
SELECT
    invoice_id,
    cents_to_dollars(amount_cents) AS amount,
    currency_code
FROM invoices;

-- ❌ BAD: Ambiguous naming
CREATE TABLE bad_table (
    price INTEGER  -- Price dalam apa? Dollars? Cents? Unclear!
);
```

### f. SOLUSI #2: Store sebagai NUMERIC

**Konsep:**
Simpan sebagai `NUMERIC` dengan precision dan scale yang sesuai kebutuhan.

**Basic approach:**

```sql
-- Standard untuk most currencies (2 decimal places)
CREATE TABLE products (
    product_name TEXT,
    price NUMERIC(10, 2)  -- Max 99,999,999.99
);

-- Insert
INSERT INTO products VALUES
    ('Laptop', 999.99),
    ('Mouse', 25.50);

-- Query (already in correct format!)
SELECT * FROM products;
-- Result:
-- Laptop | 999.99
-- Mouse  | 25.50
```

**Precision options:**

**1. Standard retail (2 decimals):**

```sql
price NUMERIC(10, 2)
-- Max: $99,999,999.99 (almost 100 million)
-- Perfect untuk e-commerce, retail
```

**2. High-value transactions:**

```sql
price NUMERIC(15, 2)
-- Max: $9,999,999,999,999.99 (almost 10 trillion)
-- Perfect untuk B2B, enterprise, banking
```

**3. Fractional cents (4 decimals):**

```sql
price NUMERIC(10, 4)
-- Example: $100.7800
-- Untuk tax calculations, financial instruments
```

**4. Cryptocurrency:**

```sql
price NUMERIC(30, 18)
-- Support sampai 18 decimal places (Ethereum standard)
-- Bitcoin: 8 decimals, Ethereum: 18 decimals
```

**5. Unbounded (maximum flexibility):**

```sql
price NUMERIC
-- No limits, accept anything
-- Trade-off: no built-in validation
```

**Contoh lengkap:**

```sql
-- Complete money table dengan NUMERIC
CREATE TABLE transactions (
    transaction_id BIGINT PRIMARY KEY,
    amount NUMERIC(10, 2),
    currency CHAR(3),
    description TEXT,
    created_at TIMESTAMP
);

-- Insert various amounts
INSERT INTO transactions VALUES
    (1, 1234.56, 'USD', 'Payment received', NOW()),
    (2, 0.99, 'USD', 'Micro-transaction', NOW()),
    (3, 999999.99, 'USD', 'Large payment', NOW());

-- Math operations (perfectly precise!)
SELECT
    SUM(amount) AS total,
    AVG(amount) AS average,
    MAX(amount) AS largest
FROM transactions;
-- Result: All exact, no floating-point errors!
```

**Keuntungan NUMERIC approach:**

**1. Perfect Precision:**

```sql
-- Absolutely no precision loss
SELECT 100.33::NUMERIC / 3;
-- Result: 33.443333333333333333... (exact!)

-- Dibanding FLOAT
SELECT 100.33::FLOAT8 / 3;
-- Result: 33.44333333333334 (slight error)
```

**2. Flexibility:**

```sql
-- Bisa adjust precision per kebutuhan
NUMERIC(10, 2)  -- Standard
NUMERIC(10, 4)  -- More precision
NUMERIC(30, 18) -- Crypto
NUMERIC         -- Unbounded
```

**3. SQL-Native:**

```sql
-- No conversion needed untuk display
SELECT amount FROM transactions;
-- Shows: 100.78 (already formatted)

-- Dibanding INTEGER approach
SELECT amount_cents / 100.0 FROM transactions;
-- Extra conversion step
```

**4. Clear Intent:**

```sql
-- Self-documenting
price NUMERIC(10, 2)
-- Jelas: decimal number dengan 2 places
```

**Trade-offs:**

**✅ Pros:**

- ✅ Perfect accuracy (critical untuk money!)
- 🎯 Natural representation (100.78 ya 100.78)
- 🔧 Flexible precision (sesuaikan per kebutuhan)
- 📊 Built-in validation (precision/scale enforced)
- 🎨 No conversion untuk display

**⚠️ Cons:**

- 🐌 **Slower** than INTEGER (demonstrated: ~2x slower)
- 💾 Variable storage size (bisa lebih besar)
- ⚠️ Unbounded NUMERIC = no validation

**Performance comparison (recall dari video sebelumnya):**

```
INTEGER:  ~1.5s for 20M operations (fastest)
NUMERIC:  ~4.5s for 20M operations (slower)
Difference: ~3x slower than INTEGER

Tapi untuk money: PRECISION > SPEED!
```

**Best practices:**

```sql
-- ✅ GOOD: Explicit precision
CREATE TABLE prices (
    amount NUMERIC(10, 2) NOT NULL CHECK (amount >= 0)
);

-- ✅ GOOD: Separate currency column
CREATE TABLE multi_currency (
    amount NUMERIC(10, 2),
    currency CHAR(3) NOT NULL  -- 'USD', 'EUR', etc.
);

-- ⚠️ RISKY: Unbounded tanpa validation
CREATE TABLE risky (
    amount NUMERIC  -- No limits, bisa anything!
);

-- ✅ BETTER: Bounded with CHECK
CREATE TABLE better (
    amount NUMERIC(15, 2) CHECK (amount BETWEEN 0 AND 999999999.99)
);
```

### g. Handling Multiple Currencies

**Problem:**
Bagaimana cara store data uang untuk multiple currencies?

**RECOMMENDED: Separate Currency Column**

```sql
-- ✅ BEST PRACTICE: Amount + Currency Code
CREATE TABLE transactions (
    transaction_id BIGINT PRIMARY KEY,
    amount NUMERIC(15, 2),     -- atau INTEGER (cents)
    currency_code CHAR(3),     -- ISO 4217: 'USD', 'EUR', 'GBP'
    description TEXT
);

-- Insert berbagai currency
INSERT INTO transactions VALUES
    (1, 100.00, 'USD', 'US payment'),
    (2, 85.50, 'EUR', 'European payment'),
    (3, 75.99, 'GBP', 'UK payment');

-- Query per currency
SELECT
    SUM(amount) AS total,
    currency_code
FROM transactions
GROUP BY currency_code;

-- Result:
-- 100.00 | USD
-- 85.50  | EUR
-- 75.99  | GBP
```

**Schema design recommendations:**

**1. Jika SEMUA data ALWAYS single currency (e.g., USD):**

```sql
-- DON'T store currency column (waste of space)
CREATE TABLE us_only (
    amount NUMERIC(10, 2)
    -- Assumed: Always USD
);
```

**2. Jika support MULTIPLE currencies:**

```sql
-- DO store currency column
CREATE TABLE multi_currency (
    amount NUMERIC(10, 2) NOT NULL,
    currency_code CHAR(3) NOT NULL,  -- Required!
    CHECK (currency_code IN ('USD', 'EUR', 'GBP', 'JPY', ...))
);
```

**3. Dengan foreign key untuk validation:**

```sql
-- Currency reference table
CREATE TABLE currencies (
    code CHAR(3) PRIMARY KEY,
    name TEXT,
    decimal_places SMALLINT,  -- 2 for USD, 0 for JPY
    symbol TEXT
);

INSERT INTO currencies VALUES
    ('USD', 'US Dollar', 2, '$'),
    ('EUR', 'Euro', 2, '€'),
    ('JPY', 'Japanese Yen', 0, '¥'),
    ('BTC', 'Bitcoin', 8, '₿');

-- Transactions dengan FK
CREATE TABLE transactions (
    amount NUMERIC(15, 8),  -- Support up to 8 decimals
    currency_code CHAR(3) REFERENCES currencies(code)
);
```

**4. INTEGER approach dengan multiple currencies:**

```sql
-- Trickier: Need to know smallest unit per currency
CREATE TABLE transactions (
    amount_smallest_unit BIGINT,  -- Cents for USD, Yen for JPY
    currency_code CHAR(3)
);

-- Conversion logic per currency
-- USD: divide by 100 (cents to dollars)
-- JPY: divide by 1 (yen is already smallest)
-- BTC: divide by 100000000 (satoshi to BTC)
```

**Warning about MONEY type:**

```sql
-- ❌ NEVER rely on MONEY for multi-currency
CREATE TABLE bad_multi (
    amount MONEY  -- Which currency? Depends on lc_monetary!
);

-- DISASTER waiting to happen!
```

### h. Optional: MONEY untuk Display (Bukan Storage)

**Valid use case:**
Jika SUDAH store sebagai NUMERIC, bisa cast ke MONEY **hanya untuk presentation**.

```sql
-- Storage: NUMERIC
CREATE TABLE products (
    name TEXT,
    price NUMERIC(10, 2)
);

INSERT INTO products VALUES ('Laptop', 999.99);

-- Display: Cast to MONEY untuk formatting
SELECT
    name,
    price,
    price::MONEY AS formatted_price
FROM products;

-- Result:
-- Laptop | 999.99 | $999.99

-- Benefit: Get currency symbol dan formatting
-- Safe: Underlying storage tetap NUMERIC
```

**Pattern ini OK karena:**

- ✅ Storage tetap sebagai NUMERIC (safe)
- ✅ MONEY hanya untuk display (cosmetic)
- ✅ Locale issue tidak affect storage

**Tapi sebaiknya:**

```sql
-- ✅ BETTER: Format di application layer
-- Don't rely on database untuk presentation formatting
-- Let frontend/app handle "$999.99" vs "999,99 €"
```

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

## 4. Catatan Tambahan / Insight

### 💡 Tips Praktis

**1. Default recommendation untuk most applications:**

```sql
-- Start dengan ini (safe default)
CREATE TABLE transactions (
    amount NUMERIC(10, 2) NOT NULL CHECK (amount >= 0),
    currency CHAR(3) DEFAULT 'USD'
);
```

**2. Untuk high-performance fintech apps:**

```sql
-- Consider INTEGER approach (Stripe-style)
CREATE TABLE payments (
    amount_cents INTEGER NOT NULL CHECK (amount_cents >= 0),
    currency CHAR(3) NOT NULL
);

-- Dengan helper functions
CREATE FUNCTION format_cents(cents INTEGER, curr CHAR(3))
RETURNS TEXT AS $$
BEGIN
    RETURN (cents / 100.0)::TEXT || ' ' || curr;
END;
$$ LANGUAGE plpgsql;
```

**3. Cryptocurrency support:**

```sql
-- Bitcoin: 8 decimals
CREATE TABLE btc_transactions (
    amount_satoshi BIGINT,  -- INTEGER approach
    -- atau
    amount_btc NUMERIC(30, 8)  -- NUMERIC approach
);

-- Ethereum: 18 decimals
CREATE TABLE eth_transactions (
    amount_wei NUMERIC(78, 0),  -- INTEGER-like (wei adalah smallest)
    -- atau
    amount_eth NUMERIC(30, 18)  -- NUMERIC dengan decimals
);
```

**4. Tax calculations (fractional cents needed):**

```sql
-- Need extra precision untuk intermediate calculations
CREATE TABLE invoices (
    subtotal NUMERIC(10, 2),
    tax_rate NUMERIC(5, 4),      -- e.g., 0.0825 = 8.25%
    tax_amount NUMERIC(10, 4),   -- 4 decimals untuk precision
    total NUMERIC(10, 2)         -- Final round ke 2 decimals
);

-- Calculation
INSERT INTO invoices (subtotal, tax_rate)
VALUES (100.00, 0.0825);

UPDATE invoices
SET
    tax_amount = subtotal * tax_rate,  -- 8.2500
    total = subtotal + (subtotal * tax_rate);  -- 108.25
```

### ⚠️ Kesalahan Umum

**1. Menggunakan MONEY tanpa memahami risiko:**

```sql
-- ❌ DANGEROUS
CREATE TABLE products (
    price MONEY  -- Locale-dependent, 2 decimals only
);
```

**2. Menggunakan FLOAT untuk money:**

```sql
-- ❌ DISASTER!
CREATE TABLE orders (
    total FLOAT8  -- Precision loss = money loss!
);

-- Problem
SELECT 0.1::FLOAT8 + 0.2::FLOAT8;
-- Result: 0.30000000000000004 (not 0.3!)
```

**3. Lupa convert INTEGER saat display:**

```sql
-- ❌ BAD: Show raw cents ke user
SELECT amount_cents FROM orders;
-- Shows: 9999 (user sees "9999" instead of "$99.99"!)

-- ✅ GOOD: Convert untuk display
SELECT amount_cents / 100.0 AS amount_dollars FROM orders;
-- Shows: 99.99
```

**4. Inconsistent currency handling:**

```sql
-- ❌ BAD: Mixing currencies tanpa identifier
INSERT INTO transactions VALUES (100.00);  -- USD atau EUR?
INSERT INTO transactions VALUES (85.50);   -- ???

-- ✅ GOOD: Always specify
INSERT INTO transactions VALUES (100.00, 'USD');
INSERT INTO transactions VALUES (85.50, 'EUR');
```

**5. Unbounded NUMERIC tanpa validation:**

```sql
-- ⚠️ RISKY
CREATE TABLE prices (
    amount NUMERIC  -- Could be 999999999999999.999999999
);

-- ✅ BETTER: Bounded dengan CHECK
CREATE TABLE prices (
    amount NUMERIC(10, 2) CHECK (amount BETWEEN 0.01 AND 99999999.99)
);
```

### 🎯 Analogies & Mental Models

**Analogi penyimpanan uang:**

```
MONEY type = Brankas dengan label yang bisa ganti-ganti
├─ Isi: $100 USD
├─ Label sekarang: "US Dollars"
├─ Besok label berubah: "British Pounds"
└─ Problem: Isi tidak berubah, tapi label misleading!

INTEGER (cents) = Menyimpan koin receh
├─ Store: 10078 koin (cents)
├─ Display: "Ini $100.78" (manual conversion)
├─ Pro: Cepat hitung koin
└─ Con: Harus convert setiap tampil

NUMERIC = Menyimpan pas seperti yang tertulis
├─ Store: $100.78
├─ Display: $100.78 (no conversion)
├─ Pro: What you see is what you store
└─ Con: Sedikit lebih lambat proses
```

**Analogi Stripe API:**

```
Stripe uses INTEGER approach:
$50.00 → Send sebagai 5000 (cents)
$0.99  → Send sebagai 99 (cents)

Why?
- Integers mudah serialize
- Tidak ada ambiguity
- Fast network transmission
- No precision loss dalam transit
```

### 📊 Real-World Patterns

**Pattern 1: E-commerce simple (single currency):**

```sql
CREATE TABLE products (
    product_id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC(8, 2) NOT NULL CHECK (price > 0),
    -- Assumed: USD only
    stock INTEGER DEFAULT 0
);

CREATE TABLE orders (
    order_id BIGSERIAL PRIMARY KEY,
    user_id BIGINT,
    subtotal NUMERIC(10, 2),
    tax NUMERIC(10, 2),
    shipping NUMERIC(10, 2),
    total NUMERIC(10, 2)
    -- All USD
);
```

**Pattern 2: Multi-currency marketplace:**

```sql
CREATE TABLE currencies (
    code CHAR(3) PRIMARY KEY,
    name TEXT,
    symbol TEXT,
    decimals SMALLINT
);

CREATE TABLE products (
    product_id BIGSERIAL PRIMARY KEY,
    name TEXT,
    price NUMERIC(12, 4),  -- Extra precision
    currency_code CHAR(3) REFERENCES currencies(code)
);

CREATE TABLE orders (
    order_id BIGSERIAL PRIMARY KEY,
    items_total NUMERIC(15, 4),
    currency_code CHAR(3),
    exchange_rate NUMERIC(10, 6),  -- For conversion
    base_currency_total NUMERIC(15, 4)  -- Converted to base
);
```

**Pattern 3: Fintech (Stripe-inspired):**

```sql
CREATE TABLE transactions (
    id BIGSERIAL PRIMARY KEY,
    amount_cents INTEGER NOT NULL,  -- Smallest unit
    currency CHAR(3) NOT NULL,
    type TEXT CHECK (type IN ('charge', 'refund', 'payout')),
    status TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Helper view untuk human-readable
CREATE VIEW transactions_display AS
SELECT
    id,
    amount_cents / 100.0 AS amount,
    currency,
    type,
    status,
    created_at
FROM transactions;
```

**Pattern 4: Cryptocurrency exchange:**

```sql
CREATE TABLE crypto_balances (
    user_id BIGINT,
    currency TEXT,  -- 'BTC', 'ETH', etc.
    balance_smallest_unit NUMERIC(78, 0),  -- Support huge numbers

    -- BTC: satoshi (10^-8)
    -- ETH: wei (## 10^-18)
    decimals SMALLINT  -- 8 for BTC, 18 for ETH
);

-- Conversion functions
CREATE FUNCTION satoshi_to_btc(satoshi NUMERIC)
RETURNS NUMERIC(30, 8) AS $$
BEGIN
    RETURN satoshi / 100000000.0;
END;
$$ LANGUAGE plpgsql;

CREATE FUNCTION wei_to_eth(wei NUMERIC)
RETURNS NUMERIC(30, 18) AS $$
BEGIN
    RETURN wei / 1000000000000000000.0;
END;
$$ LANGUAGE plpgsql;
```

### 🔍 Deep Dive: Kenapa MONEY Type Dibuat?

**Historical context:**

PostgreSQL menciptakan `MONEY` type untuk:

1. **Convenience:** Easy currency formatting
2. **Performance:** Faster than arbitrary precision (stored as 8-byte integer internally)
3. **Legacy compatibility:** Match dengan database lain yang punya MONEY type

**Kenapa sekarang tidak recommended:**

1. **Global economy changed:**

   - Multi-currency applications adalah norm (bukan exception)
   - Cryptocurrency butuh precision > 2 decimals
   - International commerce membutuhkan explicit currency tracking

2. **Better alternatives exist:**

   - NUMERIC memberikan perfect precision
   - INTEGER approach (popularized by Stripe) memberikan performance
   - Application-layer formatting lebih flexible

3. **Locale dependency adalah design flaw:**
   - Database seharusnya store data, bukan presentation
   - `lc_monetary` bisa berubah, data storage tidak boleh berubah
   - Mixing storage dan presentation = violation of separation of concerns

**Video quote:**

> "I would not rely on money at all for storage."

### 📈 Performance Considerations

**Benchmark comparison (illustrative):**

```
Operation: SUM of 10 million rows

INTEGER (cents):    ~500ms  ⚡⚡⚡ FASTEST
MONEY:              ~800ms  ⚡⚡ Fast
NUMERIC(10,2):      ~1200ms 🐌 Slower
NUMERIC (unbounded): ~1500ms 🐌 Slowest

INSERT 1 million rows:

INTEGER:  ~2s   ⚡⚡⚡
MONEY:    ~2.5s ⚡⚡
NUMERIC:  ~3.5s 🐌
```

**When performance matters:**

- High-frequency trading systems → INTEGER
- Payment processors (Stripe, PayPal) → INTEGER
- Regular e-commerce → NUMERIC is fine
- Reporting/analytics → NUMERIC is fine

**Storage size comparison:**

```
Value: $100.78

INTEGER (cents):     4 bytes  (INT4: 10078)
MONEY:               8 bytes  (fixed)
NUMERIC(10,2):       ~6-8 bytes (variable)
NUMERIC (unbounded): variable (could be larger)
```

**Tradeoff:**

- INTEGER: Smallest + fastest, tapi butuh conversion logic
- NUMERIC: Sedikit lebih besar + slower, tapi natural representation

### 🛠️ Migration Strategies

**Jika sudah pakai MONEY, cara migrate:**

**Strategy 1: Migrate ke NUMERIC:**

```sql
-- Step 1: Add new column
ALTER TABLE products
ADD COLUMN price_numeric NUMERIC(10, 2);

-- Step 2: Copy data
UPDATE products
SET price_numeric = price::NUMERIC;

-- Step 3: Verify
SELECT
    price AS old_money,
    price_numeric AS new_numeric,
    price::NUMERIC = price_numeric AS matches
FROM products;

-- Step 4: Drop old, rename new
ALTER TABLE products DROP COLUMN price;
ALTER TABLE products RENAME COLUMN price_numeric TO price;
```

**Strategy 2: Migrate ke INTEGER:**

```sql
-- Step 1: Add new column
ALTER TABLE products
ADD COLUMN price_cents INTEGER;

-- Step 2: Copy data (multiply by 100)
UPDATE products
SET price_cents = (price::NUMERIC * 100)::INTEGER;

-- Step 3: Add currency column jika multi-currency
ALTER TABLE products
ADD COLUMN currency CHAR(3) DEFAULT 'USD';

-- Step 4: Verify
SELECT
    price AS old_money,
    price_cents AS new_cents,
    price_cents / 100.0 AS converted_back
FROM products;

-- Step 5: Drop old, rename new
ALTER TABLE products DROP COLUMN price;
ALTER TABLE products RENAME COLUMN price_cents TO price_cents;
```

### 🧪 Testing Your Understanding

**Quiz mental:**

1. **Kenapa MONEY type tidak recommended?**

   - Jawaban: (1) Hanya 2 decimal places (2) Locale-dependent display (misleading)

2. **Kapan gunakan INTEGER untuk money?**

   - Jawaban: High-performance apps, API integrations (Stripe-style), single currency

3. **Kapan gunakan NUMERIC untuk money?**

   - Jawaban: Multi-currency, need clarity, fractional cents, crypto

4. **Jika ada multi-currency, apa yang harus disimpan?**

   - Jawaban: Amount + separate currency_code column

5. **Bolehkah pakai FLOAT untuk money?**

   - Jawaban: ❌ NEVER! Precision loss = money loss

6. **$100.78 disimpan sebagai INTEGER, value berapa?**

   - Jawaban: 10078 (cents)

7. **Stripe menyimpan $50.00 sebagai apa?**
   - Jawaban: 5000 (integer dalam smallest unit)

### 🎓 Advanced: Custom DOMAIN Type

**Create custom money domain:**

```sql
-- Custom domain untuk USD (NUMERIC approach)
CREATE DOMAIN usd_amount AS NUMERIC(10, 2)
    CHECK (VALUE >= 0);

-- Usage
CREATE TABLE products (
    name TEXT,
    price usd_amount  -- Custom domain type
);

-- Validation automatic
INSERT INTO products VALUES ('Laptop', 999.99);  -- ✅ OK
INSERT INTO products VALUES ('Mouse', -10.00);   -- ❌ ERROR: violates check

-- Benefits:
-- 1. Reusable across tables
-- 2. Centralized validation
-- 3. Self-documenting
```

**Custom domain untuk cents (INTEGER approach):**

```sql
-- Custom domain untuk cents
CREATE DOMAIN usd_cents AS INTEGER
    CHECK (VALUE >= 0);

CREATE TABLE products (
    name TEXT,
    price_cents usd_cents
);

-- Helper functions
CREATE FUNCTION cents_to_usd(cents usd_cents)
RETURNS NUMERIC(10,2) AS $$
BEGIN
    RETURN cents / 100.0;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

CREATE FUNCTION usd_to_cents(dollars NUMERIC)
RETURNS usd_cents AS $$
BEGIN
    RETURN (dollars * 100)::INTEGER;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Usage
SELECT cents_to_usd(10078);  -- Returns: 100.78
SELECT usd_to_cents(100.78); -- Returns: 10078
```

**Multi-currency domain:**

```sql
-- Base domain
CREATE DOMAIN currency_code AS CHAR(3)
    CHECK (VALUE ~ '^[A-Z]{3}$');  -- Must be 3 uppercase letters

CREATE DOMAIN money_amount AS NUMERIC(15, 4)
    CHECK (VALUE >= 0);

-- Table
CREATE TABLE transactions (
    amount money_amount NOT NULL,
    currency currency_code NOT NULL
);

-- Enforces format automatically
INSERT INTO transactions VALUES (100.50, 'USD');  -- ✅
INSERT INTO transactions VALUES (100.50, 'usd');  -- ❌ Not uppercase
INSERT INTO transactions VALUES (100.50, 'US');   -- ❌ Not 3 chars
INSERT INTO transactions VALUES (-10, 'USD');     -- ❌ Negative
```

### 📚 Summary Tables

**Decision matrix berdasarkan use case:**

```
╔═══════════════════════════╦═══════════════════════╦═══════════════╗
║ Use Case                  ║ Recommended Type      ║ Why           ║
╠═══════════════════════════╬═══════════════════════╬═══════════════╣
║ Simple e-commerce (USD)   ║ NUMERIC(10, 2)        ║ Clear, safe   ║
║ High-volume payments      ║ INTEGER (cents)       ║ Performance   ║
║ Multi-currency retail     ║ NUMERIC(12, 2) + curr ║ Flexibility   ║
║ Financial instruments     ║ NUMERIC(15, 4)        ║ Precision     ║
║ Cryptocurrency            ║ NUMERIC(30, 18)       ║ Many decimals ║
║ Tax calculations          ║ NUMERIC(10, 4)        ║ Intermediate  ║
║ API integrations (Stripe) ║ INTEGER (cents)       ║ Convention    ║
║ Accounting system         ║ NUMERIC(15, 2)        ║ Audit trail   ║
║ International B2B         ║ NUMERIC(15, 4) + curr ║ Multi-curr    ║
╚═══════════════════════════╩═══════════════════════╩═══════════════╝
```

**Comparison: INTEGER vs NUMERIC for money:**

```
╔═══════════════════╦═══════════════════╦═══════════════════╗
║ Aspect            ║ INTEGER (cents)   ║ NUMERIC(10, 2)    ║
╠═══════════════════╬═══════════════════╬═══════════════════╣
║ Performance       ║ ⚡⚡⚡ Fastest     ║ 🐌 ~2-3x slower   ║
║ Storage size      ║ 4 bytes (fixed)   ║ 6-8 bytes (var)   ║
║ Precision         ║ ✅ Perfect        ║ ✅ Perfect        ║
║ Natural repr      ║ ❌ Need convert   ║ ✅ Direct         ║
║ Display logic     ║ ⚠️ App must handle║ ✅ DB handles     ║
║ Type safety       ║ ✅ Strong         ║ ✅ Strong         ║
║ API-friendly      ║ ✅ Yes (Stripe)   ║ ⚠️ Need serialize ║
║ Multi-currency    ║ ⚠️ Complex logic  ║ ✅ Straightforward║
║ Learning curve    ║ ⚠️ Steeper        ║ ✅ Gentler        ║
║ Real-world use    ║ Stripe, PayPal    ║ Most e-commerce   ║
╚═══════════════════╩═══════════════════╩═══════════════════╝
```

**What to NEVER do:**

```
❌ NEVER: Use FLOAT/REAL for money
❌ NEVER: Rely on MONEY type untuk production
❌ NEVER: Mix currencies tanpa identifier
❌ NEVER: Assume 2 decimals cukup untuk semua currency
❌ NEVER: Forget validation (CHECK constraints)
```

**What to ALWAYS do:**

```
✅ ALWAYS: Use NUMERIC atau INTEGER untuk money
✅ ALWAYS: Add currency_code column jika multi-currency
✅ ALWAYS: Validate amounts (CHECK amount >= 0)
✅ ALWAYS: Document dalam kode: "stored in cents" atau "2 decimals"
✅ ALWAYS: Test rounding behavior
✅ ALWAYS: Consider timezone untuk transactions
```

## 5. Kesimpulan

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
