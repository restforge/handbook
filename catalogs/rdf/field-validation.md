# Field Validasi (`fieldValidation`)

Array berisi aturan validasi per-field. Diterapkan pada endpoint `/create` dan `/update` saja, tidak pada `/import`, `/datatables`, `/first`, atau `/delete`.

```json
{
    "fieldValidation": [
        {
            "name": "category_id",
            "type": "string",
            "constraints": {
                "primaryKey": true,
                "autoGenerate": true,
                "required": true,
                "maxLength": 36
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
| `date` | DATE | `"2026-01-01"` |
| `timestamp` | TIMESTAMP, tanpa zona (zona aplikasi `TIMEZONE` diterapkan saat tulis/baca, lihat [`datetime-fields.md`](./datetime-fields.md#konfigurasi-zona-dan-format)) | `"2026-01-16 08:30:45.123"` |
| `timestamptz` | TIMESTAMPTZ, momen berzona. Hanya PostgreSQL | `"2026-01-16T08:30:45.123+07:00"` |
| `datetime` | Alias legacy `timestamp`, diterima sebagai masukan; generator selalu meng-emit `timestamp` | `"2026-01-16 08:30:45"` |
| `time` | TIME | `"14:30:00"` |
| `uuid` | UUID | `"018f1a3b-7c9d-7abc-9def-0123456789ab"` (UUID v7, RFC 9562) |
| `json` | JSON, JSONB | `{"key": "value"}` |
| `array` | ARRAY | `[1, 2, 3]` |

### Catatan Tipe `timestamp`, `timestamptz`, dan `datetime`

`timestamp` adalah tipe kanonik untuk kolom tanggal-waktu tanpa zona, selaras dengan sistem tipe
SDF (SDF `timestamp` menghasilkan DDL `TIMESTAMP`). `timestamptz` adalah tipe terpisah untuk momen
absolut berzona, selaras dengan SDF `timestamptz` (DDL `TIMESTAMPTZ`, hanya PostgreSQL). `datetime`
tetap diterima sebagai alias legacy dari `timestamp`: runtime memperlakukannya identik, dan kolom
`DATETIME` MySQL dinormalisasi ke `timestamp` saat introspeksi. Generator (`payload generate`)
meng-emit `timestamp` untuk kolom tanpa zona dan `timestamptz` untuk kolom berzona, baik di
`fieldValidation` maupun di [`dateTimeFields`](./datetime-fields.md). Untuk pemrosesan nilai waktu
(normalisasi input, pemformatan output, konfigurasi zona), lihat [`datetime-fields.md`](./datetime-fields.md).

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
| `nullable` | boolean | Boleh NULL (default: `true`). `false` pada field yang tidak `required` menolak nilai kosong yang dikirim (lihat [Nilai Kosong dan `nullable`](#nilai-kosong-dan-nullable)) |
| `primaryKey` | boolean | Penanda primary key |
| `autoGenerate` | boolean | Auto-generate value oleh runtime saat INSERT bila field kosong. Tipe yang didukung: `uuid`, `string`, `date`, `datetime`, `timestamp`, `timestamptz` (lihat tabel di bawah) |
| `filledByHook` | boolean | Nilai field diisi handler `onBefore*`, sehingga `required` tidak dicek terhadap request body (lihat [Field yang Diisi Handler](#field-yang-diisi-handler-filledbyhook)) |

## Constraint Spesifik per Tipe

| Tipe | Constraint Spesifik |
|------|---------------------|
| `string`, `text` | `minLength`, `maxLength`, `pattern`, `patternMessage`, `format` (`email`/`phone`/`url`/`uuid`), `enum`, `notEqual`, `trim`, `lowercase`, `uppercase`, `abbreviation` (khusus `string`, lihat [Kode Singkatan](#kode-singkatan-abbreviation)). `text` divalidasi identik dengan `string`; bedanya generator tidak menurunkan `maxLength` otomatis untuk `text` (unbounded). |
| `integer`, `decimal`, `number` | `min`, `max`, `precision`, `scale`, `positive`, `negative`, `integer`, `notEqual`, `format` (`currency`) |
| `date`, `datetime`, `timestamp`, `timestamptz` | `min`, `max`, `before`, `after`. `format` ditolak `payload validate` karena pola tanggal selalu diambil dari `DATEFORMAT`/`DATETIMEFORMAT` (lihat [`datetime-fields.md`](./datetime-fields.md#satu-standar-per-aplikasi)) |
| `boolean` | `default`, `strict` |
| `array` | `minItems`, `maxItems`, `uniqueItems` |
| `json` | `schema` (opsional) |

## `precision`, `scale`, dan `format` pada Field Numerik

Pada tipe `integer`, `decimal`, dan `number`, `precision` adalah jumlah digit total dan `scale` adalah jumlah digit desimal, yaitu padanan langsung SDF `decimal:M,N`, dengan `precision` = M dan `scale` = N. `payload generate` mengisi keduanya dari skala kolom database sumbernya; kolom berskala 0 dan tipe database tanpa skala deklaratif (mis. SQLite tanpa `DECIMAL(M,N)`) tidak mendapat kedua key ini sama sekali.

Runtime menolak nilai yang jumlah digit desimalnya melebihi `scale`, misalnya `1.234` pada kolom `decimal:20,2`. `precision` hanya mencatat definisi kolom dan tidak divalidasi runtime.

Constraint `format` pada tipe numerik menerima satu nilai sah, `"currency"`. Setelah `payload migrate`, nilai ini menjadi atribut field UDF `format: "currency"`; runtime tidak memvalidasi constraint ini, pengaruhnya murni pada tampilan UDF (lihat [`catalogs/udf/field-attributes.md`](../udf/field-attributes.md#format-dan-decimalplaces-untuk-field-number)). Nilai selain `"currency"` ditolak `payload validate`:

```
fieldValidation['<name>'].constraints.format '<value>' in <fileName> is not supported for numeric fields; the only valid value is "currency"
```

## Turunan CHECK SDF

`payload generate` dan `payload sync` menurunkan [`checks` SDF](../sdf/check-constraints.md) ke `constraints` field yang dirujuk. SDF dibaca dari `--schema-path` (default folder `schema`).

| CHECK SDF | Constraint RDF |
|-----------|----------------|
| `in` pada `string` | `enum` |
| `gte: N` | `min: N` |
| `lte: N` | `max: N` |
| `gt: N` | `min` bernilai N ditambah satu langkah terkecil kolom |
| `lt: N` | `max` bernilai N dikurangi satu langkah terkecil kolom |
| `eq` pada `string` | `enum` berisi satu nilai |
| `eq` pada numerik | `min` dan `max` bernilai sama |
| `neq` | `notEqual` |

Langkah terkecil kolom adalah 1 untuk `integer` dan `bigint`, dan `10^-scale` untuk `decimal:M,N`. Contohnya, `{ field: 'deposit_value', gt: 0 }` pada `decimal:20,2` menjadi `min: 0.01`. CHECK pada kolom tanggal dan waktu tidak diturunkan.

Constraint yang sudah ada di entry tidak ditimpa, sehingga nilai yang ditulis manual tetap berlaku. Generator juga menulis registry [`checkConstraints`](./check-constraints.md) untuk memetakan pelanggaran CHECK di database ke field.

```json
{
    "name": "deposit_value",
    "type": "number",
    "constraints": {
        "scale": 2,
        "precision": 20,
        "required": true,
        "min": 0.01
    }
}
```

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
| `timestamptz` | ISO 8601 timestamp current | `new Date().toISOString()` |
| `datetime` | ISO 8601 timestamp current | `new Date().toISOString()` |
| `date` | Tanggal hari ini menurut `TIMEZONE` (YYYY-MM-DD) | Tanggal kalender zona aplikasi, bukan tanggal UTC |
| Tipe lain (`integer`, `decimal`, `boolean`, dst) | Tidak didukung | Constraint diabaikan |

**Catatan UUID v7**: RESTForge v4 menggunakan UUID v7 (RFC 9562), bukan v4. UUID v7 mengandung Unix timestamp di prefix sehingga lexicographically sortable berdasarkan waktu generate. Format: `018f1a3b-7c9d-7abc-9def-0123456789ab` (karakter ke-13 selalu `7`). Keuntungan dibanding v4:

| Aspek | UUID v4 | UUID v7 |
|-------|---------|---------|
| Sortability | Acak (random) | Sortable berdasarkan waktu generate |
| B-tree index performance | Buruk (random insert menyebabkan page split) | Baik (sequential insert seperti auto-increment) |
| Privacy | Tidak ada informasi waktu | Mengandung timestamp (dapat di-decode) |
| Format | `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx` | `xxxxxxxx-xxxx-7xxx-yxxx-xxxxxxxxxxxx` |

**Konvensi penulisan payload**: kolom UUID di SDF ditulis `string:36`, sehingga `payload generate` menurunkan primary key UUID sebagai tipe `string`:

```json
{ "name": "category_id", "type": "string", "constraints": { "primaryKey": true, "autoGenerate": true, "required": true, "maxLength": 36 } }
```

Tipe `uuid` hanya muncul bila primary key di database bertipe UUID native PostgreSQL, misalnya pada database lama yang di-introspect. Kedua tipe menghasilkan UUID v7 yang sama:

```json
{ "name": "category_id", "type": "uuid",   "constraints": { "primaryKey": true, "autoGenerate": true } }
```

Detail lengkap setiap constraint, format error message, contoh per tipe, dan kompatibilitas mundur dengan struktur lama (`fieldName` saja tanpa `fieldValidation`) dibahas di dokumentasi fitur field validation terpisah.

## Field yang Diisi Handler (`filledByHook`)

Sebagian field wajib dihitung server, misalnya nomor dokumen atau tanggal jatuh tempo, lewat handler [component engine](../../features/component-engine/README.md#mengisi-nilai-field-dari-handler) yang mengembalikan `{ success: true, data: { ... } }`. Field seperti ini diberi `"filledByHook": true` di samping `"required": true`.

```json
{ "name": "jatuh_tempo", "type": "date", "constraints": { "required": true, "filledByHook": true } }
```

Validasi request tidak menuntut nilai field tersebut dari client. Setelah handler `onBefore*` selesai, aturan `required` diperiksa lagi. Field yang tetap kosong ditolak dengan HTTP 400 `VALIDATION_FAILED` dan pesan berikut di `details`.

```
Field 'jatuh_tempo' is required but was not filled by the onBeforeInsert hook
```

Payload yang memakai `filledByHook` wajib mendaftarkan handler `onBeforeInsert`, `onBeforeUpdate`, `onBeforeCompositeInsert`, atau `onBeforeCompositeUpdate` di `components`. Tanpa handler tersebut, validasi payload menolak payload. Nilai `filledByHook` harus boolean.

Di frontend hasil [`payload migrate`](../../commands/restforge-backend/payload/migrate.md#field-yang-diisi-hook-filledbyhook), field ini tampil `readonly` dan tidak wajib diisi di form.

## Kode Singkatan (`abbreviation`)

Constraint `abbreviation` membentuk kode singkatan dari field teks lain saat insert, misalnya `LPK` untuk "Laptop & Komputer". Constraint ini diturunkan `payload generate` dan `payload sync` dari blok [`abbreviations`](../sdf/abbreviations.md) di SDF, dan ditulis ulang dari SDF setiap kali `payload sync` dijalankan. Perubahan manual di RDF tidak dipertahankan.

```json
{
  "name": "product_category_code",
  "type": "string",
  "constraints": {
    "required": true,
    "maxLength": 32,
    "unique": true,
    "abbreviation": {
      "from": "product_category_name",
      "maxLength": 3
    }
  }
}
```

| Properti | Tipe | Keterangan |
|----------|------|-----------|
| `from` | string | Field sumber. Harus ada di `fieldValidation` dan berbeda dari field target |
| `maxLength` | integer | Panjang maksimal kode, antara 1 dan `maxLength` field target |
| `ignoreWords` | array of string | Kata tambahan yang dilewati, diturunkan dari SDF bila ditulis |
| `scope` | array of string | Kolom lain pada unique gabungan field target, misalnya `["tenant_id"]` pada project multi-tenant. Bentrok hanya dihitung di antara baris dengan nilai kolom yang sama |

Field target harus bertipe `string` dan bertanda `"unique": true` atau punya `scope`. Validasi payload menolak constraint yang `from`-nya tidak ada, `maxLength`-nya di luar batas, atau `ignoreWords`-nya berisi karakter selain huruf dan angka. Aturan yang sama berlaku untuk `fieldValidation` di `masterDetail`.

Saat insert, runtime menjalankan constraint ini dengan urutan berikut.

1. Validasi request tidak menuntut nilai field target, meskipun field itu `required`.
2. Handler `onBeforeInsert` atau `onBeforeCompositeInsert` berjalan lebih dulu, sehingga handler masih bisa mengisi kode sendiri.
3. Bila field target masih kosong, runtime membentuk kode dari field sumber lalu memeriksa bentrok ke tabel, misalnya `AKK`, kemudian `AK2` bila `AKK` sudah dipakai.
4. Bila field target sudah diisi request atau handler, nilai itu disimpan apa adanya tanpa diberi angka.
5. Bila dua insert bersamaan memakai kode yang sama, insert yang kalah dihitung ulang dan dicoba lagi paling banyak lima kali.

Constraint ini tidak membentuk ulang kode saat update. Request update tetap boleh mengirim kode baru, dengan aturan panjang dan unique yang sama. Jalur ini tidak memakai Redis, sehingga tetap berjalan saat `IDGEN_ENABLED=false`.

| Kondisi | Status | Pesan di `details` |
|---------|--------|---------------------|
| Nilai kiriman lebih panjang dari `abbreviation.maxLength` | 400 `VALIDATION_FAILED` | `Field 'product_category_code' must not exceed 3 characters` |
| Field sumber kosong atau tidak berisi huruf dan angka | 400 `VALIDATION_FAILED` | `Field 'product_category_code' cannot be generated: 'product_category_name' has no letters or digits` |
| Semua angka untuk kode dasar sudah terpakai | 409 `ABBREVIATION_EXHAUSTED` | `Field 'product_category_code' has no free code left for base 'AKK' within maxLength 3` |
| Nilai kiriman sudah dipakai baris lain | 409 `UNIQUE_VIOLATION` | Lihat [Format Error Response 409](#format-error-response-409-duplicate-entry) |

Di frontend hasil [`payload migrate`](../../commands/restforge-backend/payload/migrate.md#field-kode-singkatan-abbreviation), field ini tampil `readonly` dan tidak wajib diisi di form.

## Nilai Kosong dan `nullable`

Field opsional yang dikosongkan boleh dikirim sebagai `null`, sebagai string kosong `""`, atau tidak dikirim sama sekali. Field yang tidak `required` dan bernilai kosong tidak melewati validasi tipe, lalu disimpan sesuai tabel berikut.

| Nilai di request | Tipe `string`, `text` | Tipe lain (`date`, `timestamp`, `integer`, `boolean`, `uuid`, `json`, dst.) |
|------------------|-----------------------|------------------------------------------------------------------------------|
| `""` | Disimpan `""` | Disimpan NULL |
| `null` | Disimpan NULL | Disimpan NULL |
| Tidak dikirim | Create memakai default kolom, update tidak mengubah kolom | Create memakai default kolom, update tidak mengubah kolom |

Konversi `""` menjadi NULL membuat form yang mengosongkan input tanggal atau angka tetap tersimpan, karena database menolak `""` pada kolom bertipe tanggal, angka, dan boolean. Oracle menyimpan `""` pada kolom string sebagai NULL karena sifat VARCHAR2.

Constraint `nullable` mengatur boleh tidaknya field opsional bernilai NULL. Tanpa `nullable`, atau dengan `"nullable": true`, nilai kosong disimpan seperti tabel di atas. Dengan `"nullable": false`, request yang mengirim `null`, atau `""` pada tipe selain `string` dan `text`, ditolak dengan HTTP 400 `VALIDATION_FAILED`.

```json
{ "name": "approved_on", "type": "date", "constraints": { "nullable": false } }
```

```
Field 'approved_on' cannot be null
```

Field `nullable: false` yang tidak dikirim tetap diterima, sehingga create memakai default kolom dan update tidak mengubah kolomnya. Untuk field yang wajib selalu dikirim, pakai `required`. Pada field `required`, nilai kosong sudah ditolak oleh `required`, sehingga `nullable` tidak berpengaruh.

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
| `scale` | `scaleMessage` |
| `notEqual` | `notEqualMessage` |
| `before` | `beforeMessage` |
| `after` | `afterMessage` |

## Format Error Response

Validasi yang gagal pada endpoint `/create`, `/update`, `/create-composite`, atau `/update-composite` menghasilkan HTTP status `400` dengan struktur berikut. Bentuk yang sama dipakai saat nilai dari handler `onBefore*` melanggar `fieldValidation`.

```json
{
    "success": false,
    "error": "Validation failed",
    "code": "VALIDATION_FAILED",
    "message": "Invalid data",
    "details": {
        "deposit_value": ["Field 'deposit_value' must be at least 0.01"],
        "guest_name": ["Field 'guest_name' must not exceed 100 characters"]
    },
    "errors": [
        "Field 'deposit_value' must be at least 0.01",
        "Field 'guest_name' must not exceed 100 characters"
    ],
    "timestamp": "2026-01-16T10:30:00.000Z"
}
```

Property `details` berisi map dari nama field ke array pesan error. Satu field dapat memiliki beberapa pesan error sekaligus bila beberapa constraint gagal pada nilai yang sama. Property `errors` berisi pesan yang sama dalam bentuk array string untuk klien lama.

## Format Error Response 400 (Pelanggaran Constraint Database)

Bila nilai lolos validasi aplikasi tetapi ditolak database, operasi tulis (`create`, `update`, `adjust`, `create-composite`, `update-composite`) membalas HTTP status `400`. `code` membedakan jenis pelanggaran, dan `details` menyebut field penyebabnya.

| `code` | Penyebab | Sumber field di `details` |
|--------|----------|---------------------------|
| `CHECK_VIOLATION` | CHECK constraint | Registry [`checkConstraints`](./check-constraints.md) |
| `NOT_NULL_VIOLATION` | Kolom NOT NULL diisi kosong | Error driver database |
| `FOREIGN_KEY_VIOLATION` | Data induk tidak ditemukan | Error driver (PostgreSQL, MySQL) atau registry [`foreignKeyConstraints`](./foreign-key-constraints.md) (Oracle, SQLite) |

**CHECK:**

```json
{
    "success": false,
    "error": "Constraint violation",
    "code": "CHECK_VIOLATION",
    "message": "Invalid data",
    "details": {
        "deposit_value": ["Field 'deposit_value' must be greater than 0"]
    },
    "timestamp": "2026-01-16T10:30:00.000Z"
}
```

Bila nama constraint tidak ada di registry, `details` dihilangkan dan `message` berisi `A value violates a data rule`.

**NOT NULL:**

```json
{
    "success": false,
    "error": "Constraint violation",
    "code": "NOT_NULL_VIOLATION",
    "message": "Invalid data",
    "details": {
        "people_number": ["Field 'people_number' is required"]
    },
    "timestamp": "2026-01-16T10:30:00.000Z"
}
```

**Foreign key:**

```json
{
    "success": false,
    "error": "Foreign key constraint",
    "code": "FOREIGN_KEY_VIOLATION",
    "message": "Referenced data not found",
    "details": {
        "category_id": ["Referenced data not found"]
    },
    "timestamp": "2026-01-16T10:30:00.000Z"
}
```

Bila kolom tidak dapat ditentukan, `details` dihilangkan dan status tetap `400`.

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
    "code": "UNIQUE_VIOLATION",
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
    "code": "UNIQUE_VIOLATION",
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
    "code": "UNIQUE_VIOLATION",
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

Bila validasi aplikasi lolos tetapi database menolak, API membalas `409` untuk pelanggaran UNIQUE constraint dan `400` untuk pelanggaran CHECK, NOT NULL, dan FOREIGN KEY, masing-masing dengan `details` per field (lihat [Format Error Response 400 (Pelanggaran Constraint Database)](#format-error-response-400-pelanggaran-constraint-database)). Constraint `unique` di `fieldValidation` dilewati validasinya di application layer karena diberlakukan oleh database untuk menghindari race condition antara check dan write.

Field validation **tidak** diterapkan pada endpoint `/import`. Operasi import memakai aturan tersendiri yang dideklarasikan di key `importConfig`.

---

**Lihat juga**: [`unique-constraints.md`](./unique-constraints.md) · [`check-constraints.md`](./check-constraints.md) · [`foreign-key-constraints.md`](./foreign-key-constraints.md) · [`audit-columns.md`](./audit-columns.md) · [`commands/restforge-backend/catalog/field-validation`](../../commands/restforge-backend/catalog/field-validation.md) · [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
