# Field Validasi (`fieldValidation`)

Array berisi aturan validasi per-field. Diterapkan pada endpoint `/create` dan `/update` saja, tidak pada `/import`, `/datatables`, `/first`, atau `/delete`.

```json
{
    "fieldValidation": [
        {
            "name": "category_id",
            "type": "uuid",
            "constraints": {
                "primaryKey": true,
                "autoGenerate": true
            }
        },
        {
            "name": "category_code",
            "type": "string",
            "constraints": {
                "required": true,
                "maxLength": 20,
                "unique": true,
                "uppercase": true,
                "trim": true
            }
        }
    ]
}
```

## Struktur Entry

| Property | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `name` | string | Ya | Nama field, harus ada di `fieldName` |
| `type` | enum | Ya | Tipe data field (lihat tabel di bawah) |
| `constraints` | object | Tidak | Aturan validasi tambahan |

## Tipe yang Didukung

| Type | Database Type | Contoh Value |
|------|---------------|--------------|
| `string` | VARCHAR, CHAR | `"John Doe"` |
| `text` | TEXT (PostgreSQL/MySQL/SQLite), CLOB/NCLOB (Oracle, dinormalisasi) | `"Alamat lengkap beberapa baris..."` |
| `integer` | INTEGER, INT, BIGINT | `42` |
| `decimal` | DECIMAL, NUMERIC | `19.99` |
| `number` | NUMERIC (general) | `123.45` |
| `boolean` | BOOLEAN | `true` |
| `date` | DATE | `"01/01/2026"` |
| `datetime` | TIMESTAMP | `"01/01/2026 14:30:00"` |
| `timestamp` | TIMESTAMPTZ | `"2026-01-16T08:30:45.123Z"` |
| `time` | TIME | `"14:30:00"` |
| `uuid` | UUID | `"018f1a3b-7c9d-7abc-9def-0123456789ab"` (UUID v7, RFC 9562) |
| `json` | JSON, JSONB | `{"key": "value"}` |
| `array` | ARRAY | `[1, 2, 3]` |

### Catatan Tipe `text`

`text` adalah string **unbounded** (panjang tak dibatasi). Pada operasi tulis runtime, `text` divalidasi **identik** dengan `string`, sehingga semua constraint string berlaku — perbedaannya hanya pada perlakuan panjang implisit (lihat di bawah).

Saat payload diturunkan dari introspeksi database (`migrate`/`sync`), diskriminator tipe kolom menentukan apakah entry menjadi `string` atau `text`:

- Kolom `TEXT` (PostgreSQL, MySQL, SQLite) dan `CLOB`/`NCLOB` (Oracle, dinormalisasi ke `text` saat introspeksi) menghasilkan `type: "text"`.
- Kolom `VARCHAR`/`CHAR` tetap menghasilkan `type: "string"` dan tetap menurunkan `maxLength` dari panjang kolom.

Dua sifat penting `text` saat diturunkan dari database:

1. **Tanpa `maxLength` otomatis.** Berbeda dengan `string` dari `VARCHAR`, generator tidak menurunkan `maxLength` untuk `text` karena kolom bersifat unbounded. `maxLength` hanya ada bila ditulis manual di `constraints`.
2. **Tidak di-skip walau tanpa constraint lain.** Entry `string` nullable tanpa constraint apa pun di-skip dari `fieldValidation` agar payload tidak bloat, tetapi entry `text` **tetap dipertahankan** meski `constraints` kosong, karena tipe `text` itu sendiri adalah sinyal "long text" yang first-class.

## Constraint Umum (Berlaku untuk Semua Tipe)

| Constraint | Tipe | Deskripsi |
|------------|------|-----------|
| `required` | boolean | Field wajib diisi |
| `unique` | boolean | Nilai harus unik (diberlakukan database) |
| `default` | any | Nilai default jika tidak diisi |
| `nullable` | boolean | Boleh NULL (default: `true` jika tidak `required`) |
| `primaryKey` | boolean | Penanda primary key |
| `autoGenerate` | boolean | Auto-generate value oleh runtime saat INSERT bila field kosong. Tipe yang didukung: `uuid`, `string`, `date`, `datetime`, `timestamp` (lihat tabel di bawah) |

## Constraint Spesifik per Tipe

| Tipe | Constraint Spesifik |
|------|---------------------|
| `string`, `text` | `minLength`, `maxLength`, `pattern`, `patternMessage`, `format` (`email`/`phone`/`url`/`uuid`), `enum`, `trim`, `lowercase`, `uppercase`. `text` divalidasi identik dengan `string`; bedanya generator tidak menurunkan `maxLength` otomatis untuk `text` (unbounded). |
| `integer`, `decimal`, `number` | `min`, `max`, `precision`, `scale`, `positive`, `negative`, `integer` |
| `date`, `datetime`, `timestamp` | `format`, `min`, `max`, `before`, `after` |
| `boolean` | `default`, `strict` |
| `array` | `minItems`, `maxItems`, `uniqueItems` |
| `json` | `schema` (opsional) |

## Detail Format Preset (`format`)

Constraint `format` pada field tipe `string` menerima 4 preset siap pakai. Pattern internal ditangani oleh runtime tanpa perlu menulis regex secara manual.

| Preset | Pattern Internal | Validasi |
|--------|------------------|----------|
| `email` | `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` | Alamat dengan `@` dan domain (validasi minimal, bukan RFC 5322 strict) |
| `phone` | `/^[\d\s\-\+\(\)]+$/` | Digit dengan separator opsional (spasi, hyphen, plus, parentheses) |
| `url` | `/^https?:\/\/.+/` | URL dengan scheme `http://` atau `https://` |
| `uuid` | `/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i` | UUID standar 8-4-4-4-12 hex (case-insensitive) |

Default error message: `Field '<name>' has invalid <format> format`. Override dengan key `formatMessage`.

```json
{
    "name": "email",
    "type": "string",
    "constraints": {
        "format": "email",
        "formatMessage": "Format email tidak valid"
    }
}
```

Pemilihan antara `format` dan `pattern`: gunakan `format` untuk validasi umum yang reusable (email, phone, url, uuid). Gunakan `pattern` untuk regex custom seperti kode SKU, nomor invoice, atau format internal lainnya.

## Behavior `autoGenerate` per Tipe

Saat constraint `autoGenerate: true` aktif dan request body tidak menyertakan nilai field tersebut, runtime menghasilkan value secara otomatis sesuai tipe field. Generation dilakukan di application layer (bukan database DEFAULT), sehingga format konsisten lintas dialect.

| Tipe Field | Generated Value | Mekanisme |
|------------|-----------------|-----------|
| `uuid` | UUID v7 (RFC 9562, lexicographically sortable) | `require('uuid').v7()` |
| `string` | UUID v7 (sama dengan tipe `uuid`) | `require('uuid').v7()` |
| `timestamp` | ISO 8601 timestamp current | `new Date().toISOString()` |
| `datetime` | ISO 8601 timestamp current | `new Date().toISOString()` |
| `date` | ISO 8601 date current (YYYY-MM-DD) | `new Date().toISOString().split('T')[0]` |
| Tipe lain (`integer`, `decimal`, `boolean`, dst) | Tidak didukung | Constraint diabaikan |

**Catatan UUID v7**: RESTForge v4 menggunakan UUID v7 (RFC 9562), bukan v4. UUID v7 mengandung Unix timestamp di prefix sehingga lexicographically sortable berdasarkan waktu generate. Format: `018f1a3b-7c9d-7abc-9def-0123456789ab` (karakter ke-13 selalu `7`). Keuntungan dibanding v4:

| Aspek | UUID v4 | UUID v7 |
|-------|---------|---------|
| Sortability | Acak (random) | Sortable berdasarkan waktu generate |
| B-tree index performance | Buruk (random insert menyebabkan page split) | Baik (sequential insert seperti auto-increment) |
| Privacy | Tidak ada informasi waktu | Mengandung timestamp (dapat di-decode) |
| Format | `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx` | `xxxxxxxx-xxxx-7xxx-yxxx-xxxxxxxxxxxx` |

**Konvensi penulisan payload**: untuk field primary key UUID, RESTForge v4 menerima dua deklarasi yang menghasilkan behavior identik:

```json
{ "name": "category_id", "type": "uuid",   "constraints": { "primaryKey": true, "autoGenerate": true } }
```

```json
{ "name": "category_id", "type": "string", "constraints": { "primaryKey": true, "autoGenerate": true } }
```

Keduanya menghasilkan UUID v7 yang sama. Tipe `string` umum dipakai di payload starter agar konvensi seragam lintas database (PostgreSQL native UUID, MySQL/Oracle/SQLite menyimpan UUID sebagai VARCHAR).

Detail lengkap setiap constraint, format error message, contoh per tipe, dan kompatibilitas mundur dengan struktur lama (`fieldName` saja tanpa `fieldValidation`) dibahas di dokumentasi fitur field validation terpisah.

## Custom Error Message

Setiap constraint dapat memiliki versi `<constraint>Message` untuk override pesan default:

| Constraint | Override |
|------------|----------|
| `required` | `requiredMessage` |
| `minLength` | `minLengthMessage` |
| `maxLength` | `maxLengthMessage` |
| `format` | `formatMessage` |
| `pattern` | `patternMessage` |
| `enum` | `enumMessage` |
| `min` | `minMessage` |
| `max` | `maxMessage` |
| `positive` | `positiveMessage` |
| `before` | `beforeMessage` |
| `after` | `afterMessage` |

## Format Error Response

Validasi yang gagal pada endpoint `/create` atau `/update` menghasilkan HTTP status `400` dengan struktur:

```json
{
    "success": false,
    "error": "Validation Error",
    "message": "Invalid data",
    "details": {
        "supplier_code": [
            "Field supplier_code is required"
        ],
        "email": [
            "Field email must be a valid email address"
        ]
    },
    "requestId": "...",
    "timestamp": "2026-01-16T10:30:00.000Z"
}
```

Property `details` berisi map dari nama field ke array pesan error. Satu field dapat memiliki beberapa pesan error sekaligus bila beberapa constraint gagal pada nilai yang sama.

## Format Error Response 409 (Duplicate Entry)

Pelanggaran unique constraint pada operasi tulis (`create`, `update`, `adjust`,
`create-composite`, `update-composite`) menghasilkan HTTP status `409`. Response memakai
map `details` dengan konvensi yang sama dengan validation error `400`, sehingga client
dapat memetakan satu handler error untuk kedua status. Field yang berkonflik dibawa lewat
`details`, sementara `error` dan `message` top-level tetap statis (backward compatible).

**Bentuk body — single field UNIQUE:**

```json
{
    "success": false,
    "error": "Duplicate entry",
    "message": "A record with this value already exists",
    "details": {
        "email": ["A record with this value already exists"]
    },
    "timestamp": "2026-01-16T10:30:00.000Z"
}
```

**Bentuk body — composite UNIQUE:**

Constraint pada kombinasi beberapa kolom menghasilkan beberapa key di `details`, satu per
field pembentuk constraint:

```json
{
    "success": false,
    "error": "Duplicate entry",
    "message": "A record with this value already exists",
    "details": {
        "warehouse_id": ["A record with this combination of values already exists"],
        "sku": ["A record with this combination of values already exists"]
    },
    "timestamp": "..."
}
```

**Bentuk body — fallback:**

Bila metadata field tidak dapat diekstrak dari error driver, `details` dihilangkan. Field
`success`, `error`, `message`, dan `timestamp` tetap apa adanya:

```json
{
    "success": false,
    "error": "Duplicate entry",
    "message": "A record with this value already exists",
    "timestamp": "..."
}
```

### Pesan Single dan Composite

| Kondisi | Pesan di dalam `details` |
|---------|--------------------------|
| Single field UNIQUE | `A record with this value already exists` |
| Composite UNIQUE | `A record with this combination of values already exists` |

`message` top-level **selalu statis** (`A record with this value already exists`) di semua
kasus, identik dengan baseline sebelum penambahan `details`, demi backward compatibility.
Hanya nilai di dalam `details` yang membedakan single dan composite.

### Konvensi `details`

Sama dengan format `400`: map dari nama field ke array pesan error. Client dapat memetakan
satu handler error untuk status `400` (validation error) dan `409` (duplicate entry).

### Catatan Keamanan

Nama constraint internal database **tidak** diekspos ke response. Hanya nama field logis
yang muncul sebagai key di `details`.

Sumber pemetaan `nama constraint → field` untuk dialect MySQL dan Oracle dijelaskan di
[unique-constraints.md](./unique-constraints.md). PostgreSQL dan SQLite membaca daftar
kolom langsung dari error driver saat runtime.

## Validasi Opsional

Field yang terdaftar di `fieldValidation` namun tidak memerlukan constraint tambahan dapat dideklarasikan dengan dua cara:

1. Tidak menyertakan key `constraints` sama sekali
2. Menyertakan `constraints: {}` (object kosong)

```json
{
    "name": "notes",
    "type": "string",
    "constraints": {}
}
```

Tipe field tetap divalidasi sesuai `type` yang dideklarasikan (string, number, boolean, dst.), namun tidak ada aturan tambahan yang diterapkan.

## Kompatibilitas Mundur

Payload lama yang hanya memakai `fieldName` tanpa `fieldValidation` tetap didukung. Runtime akan:

- Mengonversi setiap entry `fieldName` menjadi field tipe `string` tanpa constraints
- Tidak menerapkan validasi tambahan pada operasi `/create` dan `/update`
- Tetap menjalankan semua action endpoint secara normal

Migrasi ke struktur baru tidak wajib, namun direkomendasikan agar mendapat manfaat type-checking dan constraint validation pada operasi tulis.

## Interaksi dengan Database Constraints

Field validation di application layer berjalan **sebelum** SQL statement dieksekusi ke database. Urutan eksekusi pada operasi `/create` dan `/update`:

1. Sanitization (`trim`, `lowercase`, `uppercase`) bila constraint relevan aktif
2. Field validation per-constraint (`required`, `minLength`, `maxLength`, `pattern`, `format`, dst.)
3. Cross-field validation (constraint `before`, `after` pada tipe date)
4. SQL execution ke database

Bila validasi aplikasi lolos tetapi database menolak (contoh: pelanggaran UNIQUE constraint, FOREIGN KEY, atau CHECK constraint), API mengembalikan error sesuai pesan database. Constraint `unique` di `fieldValidation` dilewati validasinya di application layer karena diberlakukan oleh database untuk menghindari race condition antara check dan write.

Field validation **tidak** diterapkan pada endpoint `/import`. Operasi import memakai aturan tersendiri yang dideklarasikan di key `importConfig`.

---

**Lihat juga**: [`unique-constraints.md`](./unique-constraints.md) · [`audit-columns.md`](./audit-columns.md) · [`commands/restforge-backend/catalog/field-validation`](../../commands/restforge-backend/catalog/field-validation.md) · [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
