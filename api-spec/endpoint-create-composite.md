# Create Composite — Endpoint Pembuatan Data Master-Detail

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/create-composite` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `201 Created` |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Transaction** | Atomik, seluruh operasi dalam satu transaction |
| **Auto UUID** | Header PK dan setiap detail item PK |
| **Auto Calculate** | Detail: formula per-baris. Header: count, sum dari detail |
| **Konfigurasi** | `masterDetail` di payload JSON |

---

## Ikhtisar

Endpoint `/create-composite` digunakan untuk membuat data master-detail (misalnya header transaksi + item-item detail) dalam **satu transaction atomik**. Jika salah satu operasi gagal, seluruh perubahan di-rollback.

Primary key header dan setiap detail item di-generate secara otomatis sebagai UUID. Foreign key di detail items diisi secara otomatis dari primary key header. Field kalkulasi (total per baris, total header) dihitung secara otomatis berdasarkan konfigurasi.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/stock-inbound/create-composite
```

---

## Format Request

### Struktur

```json
{
  "<nama_tabel_header>": {
    "field_header_1": "value",
    "field_header_2": "value",
    "<nama_tabel_detail>": [
      { "line_number": 1, "field_detail_1": "value" },
      { "line_number": 2, "field_detail_1": "value" }
    ]
  }
}
```

- Root key **harus sesuai** dengan nama tabel header
- Detail items berupa **array** (minimal 1 item)
- Primary key header dan detail **tidak perlu dikirim** (auto UUID)
- Foreign key di detail **tidak perlu dikirim** (auto-fill dari header PK)
- Field yang dihitung secara otomatis **tidak perlu dikirim**

### Contoh Request

```json
{
  "stock_inbound": {
    "inbound_number": "INB/2026/001",
    "inbound_date": "2026-03-30",
    "warehouse_id": "e9287574-f9e5-4cef-8641-204585c74e2c",
    "supplier_id": "1d0d2937-b347-4451-83ff-e042d35e9419",
    "reference_number": "PO/2026/001",
    "notes": "Pengadaan peralatan kantor",
    "status": "draft",
    "stock_inbound_item": [
      {
        "line_number": 1,
        "item_product_id": "88573a96-8bf6-406b-88a5-a59903934541",
        "qty_received": 10,
        "uom": "pcs",
        "unit_price": 100000,
        "notes": "Keyboard wireless"
      },
      {
        "line_number": 2,
        "item_product_id": "6f1228fd-7f30-49f5-8597-26b79dfe0a1b",
        "qty_received": 20,
        "uom": "pcs",
        "unit_price": 50000,
        "notes": "Mouse optical"
      }
    ]
  }
}
```

---

## Detail Satu Baris (`one`)

RDF yang memakai [`masterDetail.details[]`](../catalogs/rdf/master-detail.md#beberapa-detail-dalam-satu-rdf-details) dapat memuat detail `one`, yaitu detail dengan paling banyak satu baris per header. Detail ini dikirim sebagai objek di key `detailId`-nya, di dalam objek header, bukan sebagai array.

```json
{
  "business_partner": {
    "partner_code": "BP-0001",
    "partner_name": "PT Komponen Prima",
    "vendor": { "vendor_code": "VEND-001", "payment_term_days": 30 }
  }
}
```

Setiap nilai yang dikirim pada key detail `one` diperlakukan sebagai berikut.

| Nilai key detail `one` | Perilaku |
|------------------------|----------|
| Key tidak dikirim | Detail tidak dibuat |
| `null` | Detail tidak dibuat |
| Objek tanpa primary key detail | Baris detail dibuat dengan primary key otomatis, dan foreign key diisi dari primary key header |
| Objek dengan primary key detail | Baris detail dibuat dengan primary key dari request |

Nilai selain objek dan `null`, misalnya string atau array, ditolak dengan `400`.

Response memuat detail `one` sebagai objek berisi field di `detailConfig.fieldName` ditambah primary key dan foreign key detail, atau `null` bila detail tidak dibuat. Pada contoh di atas, `data.vendor` berisi objek vendor dan `data.customer` bernilai `null`. Seluruh operasi berjalan dalam satu transaction, sehingga kegagalan di salah satu detail membatalkan header dan detail lain.

### Key Error `{detailId}.{kolom}`

Dua detail `one` dapat punya kolom bernama sama, misalnya `payment_term_days` di vendor dan di customer. Karena itu pelanggaran constraint pada detail `one` memakai key `{detailId}.{kolom}` di map `details`, sehingga client dapat mengarahkan pesan ke detail yang benar. Field header dan detail `many` tetap memakai format error yang sudah ada.

```json
{
  "success": false,
  "error": "Constraint violation",
  "code": "CHECK_VIOLATION",
  "message": "Invalid data",
  "details": {
    "customer.payment_term_days": ["Field 'payment_term_days' must be at least 0"]
  },
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

| HTTP Status | Kondisi | Contoh key di `details` |
|:-----------:|---------|------------------|
| 400 | Nilai key detail `one` bukan objek dan bukan `null`, dengan pesan `Property "vendor" must be an object or null` | Tanpa `details` |
| 400 | CHECK atau NOT NULL pada kolom detail (`CHECK_VIOLATION` atau `NOT_NULL_VIOLATION`) | `customer.payment_term_days` |
| 409 | Unique constraint pada kolom detail (`UNIQUE_VIOLATION`) | `vendor.vendor_code` |

---

## Body Options (Opsional)

Endpoint `/create-composite` mendukung format request body alternatif dengan wrapper `{data, options}`. Format ini memungkinkan pengiriman opsi tambahan yang dapat dibaca oleh component engine handler dan processor.

### Format dengan Options

```json
{
  "data": {
    "stock_inbound": {
      "inbound_number": "INB/2026/001",
      "inbound_date": "2026-03-30",
      "warehouse_id": "e9287574-f9e5-4cef-8641-204585c74e2c",
      "supplier_id": "1d0d2937-b347-4451-83ff-e042d35e9419",
      "status": "draft",
      "stock_inbound_item": [
        {
          "line_number": 1,
          "item_product_id": "88573a96-8bf6-406b-88a5-a59903934541",
          "qty_received": 10,
          "uom": "pcs",
          "unit_price": 100000
        }
      ]
    }
  },
  "options": {
    "send_feedback_email": true,
    "auto_approve": false
  }
}
```

### Aturan

- Body harus memiliki **tepat 2 key**: `data` dan `options`
- `data` harus berupa object (berisi root key tabel header + detail items)
- `options` harus berupa object (berisi flag/parameter tambahan)
- Setelah `data` diekstrak, struktur di dalamnya diproses sama seperti format composite standar (root key, header fields, detail array)
- Jika format ini tidak terdeteksi, body diproses sebagai format standar (backward compatible)
- Nilai `options` tersedia di component handler sebagai template variable `{options}` dan di processor melalui `req.bodyOptions`

---

## Format Response

### Response Sukses

**HTTP Status:** `201 Created`

```json
{
  "success": true,
  "message": "stock-inbound data successfully created (with detail items)",
  "data": {
    "stock_inbound_id": "d4e57872-17df-450e-b3bd-c0fac1dcb485",
    "inbound_number": "INB/2026/001",
    "inbound_date": "2026-03-30T00:00:00.000Z",
    "warehouse_id": "e9287574-...",
    "supplier_id": "1d0d2937-...",
    "total_items": 2,
    "total_qty": 30,
    "total_amount": 2000000,
    "status": "draft",
    "stock_inbound_item": [
      {
        "stock_inbound_item_id": "a1b2c3d4-...",
        "stock_inbound_id": "d4e57872-...",
        "line_number": 1,
        "item_product_id": "88573a96-...",
        "qty_received": 10,
        "uom": "pcs",
        "unit_price": 100000,
        "total_amount": 1000000
      },
      {
        "stock_inbound_item_id": "e5f6g7h8-...",
        "stock_inbound_id": "d4e57872-...",
        "line_number": 2,
        "item_product_id": "6f1228fd-...",
        "qty_received": 20,
        "uom": "pcs",
        "unit_price": 50000,
        "total_amount": 1000000
      }
    ]
  },
  "timestamp": "2026-03-30T09:00:00.000Z"
}
```

Field header mengikuti `fieldName`. Setiap item detail berisi field di `detailConfig.fieldName` ditambah primary key detail, foreign key, dan kolom `autoCalculateFields`. Kolom lain di tabel detail, termasuk kolom audit yang tidak terdaftar, tidak ikut terkirim.

### Response Error

#### 400 — Nilai Tanggal-Waktu Tidak Valid

Terjadi ketika nilai field `date`, `timestamp`, atau `timestamptz` pada header maupun item detail tidak cocok dengan pola aktif (`DATEFORMAT`/`DATETIMEFORMAT`) maupun bentuk ISO. Tanggal saja pada field `timestamp`/`timestamptz`, tanggal kalender yang tidak nyata, jam di luar rentang, dan jam lokal pada celah atau tumpang-tindih DST termasuk kasus ini.

```json
{
  "success": false,
  "error": "Invalid datetime",
  "code": "INVALID_DATETIME",
  "message": "Invalid datetime value '18/09/2026': type 'timestamp' requires a time component; a date-only value is not accepted. Expected an ISO 8601 value or format 'dd/MM/yyyy HH:mm:ss'",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

Aturan lengkap ada di [`catalogs/rdf/datetime-fields.md`](../catalogs/rdf/datetime-fields.md#format-error).

| HTTP Status | Kondisi | Contoh Message |
|:-----------:|---------|----------------|
| 400 | Root key tidak sesuai nama tabel | `Payload must have property "stock_inbound"` |
| 400 | Detail tidak ada di dalam objek header, disertai `example` bentuk body | `Property "stock_inbound_item" is missing inside "stock_inbound". Detail rows must be nested under the header object` |
| 400 | Detail dikirim sejajar header, disertai `example` bentuk body | `Property "stock_inbound_item" is missing inside "stock_inbound". Found "stock_inbound_item" at top level; move it inside "stock_inbound"` |
| 400 | Detail bukan array | `Property "stock_inbound_item" must be an array` |
| 400 | Detail array kosong | `Property "stock_inbound_item" must be a non-empty array` |
| 400 | Validasi field header gagal (`code: VALIDATION_FAILED`) | `{ details: { field: ["message"] }, errors: ["message"] }` |
| 400 | Ditolak handler component `onBeforeCompositeInsert` atau `onAfterCompositeInsert`. Data yang ditulis handler lewat `services.db` ikut dibatalkan, kecuali pada SQLite. Penolakan di `onAfterCommitCompositeInsert` tidak menghasilkan error | Pesan dari handler saja, tanpa nama event, nama fungsi, atau path file handler |
| 409 | Unique constraint violation | `A record with this value already exists` |
| 400 | Foreign key constraint (`code: FOREIGN_KEY_VIOLATION`) | `Referenced data not found`, dengan `details` per kolom bila kolom header diketahui |
| 400 | CHECK atau NOT NULL ditolak database (`code: CHECK_VIOLATION` / `NOT_NULL_VIOLATION`) | `Invalid data`, dengan `details` per field header |
| 500 | Error database lain | `An error occurred while creating stock-inbound data` |

> Pelanggaran constraint pada tabel detail `many` dibalas tanpa `details`, sedangkan detail `one` memakai key `{detailId}.{kolom}` (lihat [Detail Satu Baris](#detail-satu-baris-one)). Rincian bentuk `400` ada di
> [`catalogs/rdf/field-validation.md`](../catalogs/rdf/field-validation.md#format-error-response-400-pelanggaran-constraint-database).
>
> **Field `details` pada 409:** response unique violation (`code: UNIQUE_VIOLATION`) menyertakan map `details` =
> `{ "<kolom>": ["<pesan>"] }` yang menunjuk kolom pelanggar (untuk constraint multi-kolom,
> tiap kolom dipetakan ke `"A record with this combination of values already exists"`).
> Field ini selalu muncul bila kolom pelanggar dapat diekstrak; bila tidak, field ini tidak disertakan. Detail:
> [`README` — Duplicate entry](./README.md#409--conflict).

---

## Field Otomatis

### Field yang Di-generate

| Field | Nilai | Keterangan |
|-------|-------|------------|
| Header PK | UUIDv7 | Auto-generate |
| Detail PK | UUIDv7 per item | Auto-generate |
| Detail FK | Header PK value | Auto-fill |
| Audit columns header dan detail | Waktu saat ini dan user_id | Selalu ditulis ke database. Muncul di response hanya bila terdaftar di `fieldName` (header) atau `detailConfig.fieldName` (detail) |

### Field yang Dihitung Otomatis (Detail Level)

Dikonfigurasi via `detailConfig.autoCalculateFields`:
```json
{ "total_amount": { "type": "calculated", "formula": "qty_received * unit_price" } }
```

### Field yang Dihitung Otomatis (Header Level)

Dikonfigurasi via `headerCalculations`:
```json
{
  "total_items": { "type": "count", "source": "items.length" },
  "total_qty": { "type": "sum", "source": "items.qty_received" },
  "total_amount": { "type": "sum", "source": "items.total_amount" }
}
```

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| CRUD Composite (Deep Dive) | — | Dokumentasi lengkap master-detail |
| Field Validation | — | Validasi field header dan detail |
| Cache | — | Invalidasi cache setelah create berhasil |
| Body Options | — | Opsi tambahan via format `{data, options}` |
