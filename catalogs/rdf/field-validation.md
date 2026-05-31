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
| `name` | string | Ya | Nama field, harus exist di `fieldName` |
| `type` | enum | Ya | Tipe data field (lihat tabel di bawah) |
| `constraints` | object | Tidak | Aturan validasi tambahan |

## Tipe yang Didukung

| Type | Database Type | Contoh Value |
|------|---------------|--------------|
| `string` | VARCHAR, TEXT, CHAR | `"John Doe"` |
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

## Constraint Umum (Berlaku untuk Semua Tipe)

| Constraint | Tipe | Deskripsi |
|------------|------|-----------|
| `required` | boolean | Field wajib diisi |
| `unique` | boolean | Nilai harus unik (di-enforce database) |
| `default` | any | Nilai default jika tidak diisi |
| `nullable` | boolean | Boleh NULL (default: `true` jika tidak `required`) |
| `primaryKey` | boolean | Penanda primary key |
| `autoGenerate` | boolean | Auto-generate value oleh runtime saat INSERT bila field kosong. Tipe yang didukung: `uuid`, `string`, `date`, `datetime`, `timestamp` (lihat tabel di bawah) |

## Constraint Spesifik per Tipe

| Tipe | Constraint Spesifik |
|------|---------------------|
| `string` | `minLength`, `maxLength`, `pattern`, `patternMessage`, `format` (`email`/`phone`/`url`/`uuid`), `enum`, `trim`, `lowercase`, `uppercase` |
| `integer`, `decimal`, `number` | `min`, `max`, `precision`, `scale`, `positive`, `negative`, `integer` |
| `date`, `datetime`, `timestamp` | `format`, `min`, `max`, `before`, `after` |
| `boolean` | `default`, `strict` |
| `array` | `minItems`, `maxItems`, `uniqueItems` |
| `json` | `schema` (opsional) |

## Detail Format Preset (`format`)

Constraint `format` pada field tipe `string` menerima 4 preset siap pakai. Pattern internal di-handle oleh runtime tanpa perlu menulis regex manual.

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

Saat constraint `autoGenerate: true` aktif dan request body tidak menyertakan nilai field tersebut, runtime menghasilkan value otomatis sesuai tipe field. Generation dilakukan di application layer (bukan database DEFAULT), sehingga format konsisten lintas dialect.

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

## Kompatibilitas Mundur (Backward Compatibility)

Payload lama yang hanya memakai `fieldName` tanpa `fieldValidation` tetap didukung. Runtime akan:

- Mengkonversi setiap entry `fieldName` menjadi field tipe `string` tanpa constraints
- Tidak menerapkan validasi tambahan pada operasi `/create` dan `/update`
- Tetap menjalankan semua action endpoint secara normal

Migrasi ke struktur baru tidak wajib, namun direkomendasikan agar mendapat manfaat type-checking dan constraint validation pada operasi tulis.

## Interaksi dengan Database Constraints

Field validation di application layer berjalan **sebelum** SQL statement dieksekusi ke database. Urutan eksekusi pada operasi `/create` dan `/update`:

1. Sanitization (`trim`, `lowercase`, `uppercase`) bila constraint relevan aktif
2. Field validation per-constraint (`required`, `minLength`, `maxLength`, `pattern`, `format`, dst.)
3. Cross-field validation (constraint `before`, `after` pada tipe date)
4. SQL execution ke database

Bila validasi aplikasi pass tetapi database menolak (contoh: violation pada UNIQUE constraint, FOREIGN KEY, atau CHECK constraint), API mengembalikan error sesuai pesan database. Constraint `unique` di `fieldValidation` di-skip validation di application layer karena di-enforce oleh database untuk menghindari race condition antara check dan write.

Field validation **tidak** diterapkan pada endpoint `/import`. Operasi import memakai aturan tersendiri yang dideklarasikan di key `importConfig`.

---

**Lihat juga**: [`audit-columns.md`](./audit-columns.md) · [`commands/restforge-backend/catalog/field-validation`](../../commands/restforge-backend/catalog/field-validation.md) · [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
