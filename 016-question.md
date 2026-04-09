# Questions

<details>

<summary>Penjelasan Query CTE?<br/></summary>

```sql
WITH new_file AS (
    SELECT '\x48656c6c6f'::BYTEA AS content
)
SELECT f.*
FROM files f, new_file nf
WHERE f.content_hash = MD5(nf.content)::UUID;
```

### Gambaran Umum Query

Query ini pada intinya ingin:

> **Mencari baris di tabel `files` yang isi filenya sama dengan sebuah data biner tertentu (`Hello`) dengan cara membandingkan hash-nya.**

Alih-alih membandingkan isi file secara langsung, query ini **menggunakan hash MD5** sebagai pembanding.

### Bagian 1: CTE (<code>WITH new_file AS (...)</code>)

```sql
WITH new_file AS (
    SELECT '\x48656c6c6f'::BYTEA AS content
)
```

Ini disebut **CTE (Common Table Expression)** — semacam _tabel sementara_ yang hanya hidup selama query ini berjalan.

### Apa isi `new_file`?

```sql
'\x48656c6c6f'::BYTEA
```

- `\x` → menandakan **hexadecimal**
- `48 65 6c 6c 6f` → jika diterjemahkan ke ASCII = **"Hello"**
- `::BYTEA` → casting ke tipe data **BYTEA** (data biner)

📌 Jadi secara logika:

| content                      |
| ---------------------------- |
| `Hello` (dalam bentuk biner) |

CTE ini menghasilkan **satu baris, satu kolom**, bernama `content`.

### Bagian 2: SELECT utama

```sql
SELECT f.*
FROM files f, new_file nf
```

Di sini:

- `files f` → tabel utama (misalnya tabel penyimpanan file)
- `new_file nf` → CTE yang tadi kita buat

Penulisan seperti ini adalah **cross join gaya lama**. Karena `new_file` hanya punya **1 baris**, efeknya sama seperti:

> "Untuk setiap baris di `files`, bandingkan dengan satu data `new_file` ini."

### Bagian 3: Kondisi WHERE (bagian paling penting)

```sql
WHERE f.content_hash = MD5(nf.content)::UUID;
```

Mari kita uraikan dari **kanan ke kiri** (ini biasanya lebih mudah dipahami).

#### 1️⃣ `nf.content`

Ini adalah data biner:

```
Hello
```

#### 2️⃣ `MD5(nf.content)`

PostgreSQL menghitung **MD5 hash** dari isi biner tersebut.

- MD5 menghasilkan **32 karakter hex**, contoh:

```
8b1a9953c4611296a827abf8c47804d7
```

📌 Penting: MD5 **selalu sama** untuk input yang sama.

#### 3️⃣ `::UUID`

```sql
MD5(nf.content)::UUID
```

Artinya:

- String MD5 tadi **dipaksa dianggap sebagai UUID**
- Ini hanya masuk akal **jika kolom `content_hash` memang disimpan sebagai UUID**

Contoh UUID dari MD5:

```
8b1a9953-c461-1296-a827-abf8c47804d7
```

(PostgreSQL cukup fleksibel dalam parsing format ini)

#### 4️⃣ Perbandingan akhir

```sql
f.content_hash = MD5(nf.content)::UUID
```

Artinya:

> "Ambil semua baris dari `files` yang `content_hash`-nya **sama dengan hash MD5 dari `Hello`**."

### Kesimpulan Alur Logika

Kalau dijelaskan dengan bahasa manusia:

1. Buat data biner sementara berisi `"Hello"`
2. Hitung hash MD5 dari data tersebut
3. Ubah hash itu menjadi UUID
4. Cari file di tabel `files` yang hash-nya sama
5. Tampilkan seluruh kolom file tersebut

</details>
