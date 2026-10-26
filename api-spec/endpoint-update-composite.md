# Update Composite — Endpoint Pembaruan Data Master-Detail

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/update-composite` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `200 OK` |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Transaction** | Atomik, seluruh operasi dalam satu transaction |
| **Detail Operations** | `insert`, `update`, `delete` (dalam satu request) |
| **Urutan Eksekusi** | DELETE → UPDATE → INSERT |
| **Auto Recalculate** | Header totals dihitung ulang setelah semua operasi detail |

---

## Ikhtisar

Endpoint `/update-composite` digunakan untuk memperbarui data master-detail dalam satu transaction atomik. Berbeda dengan `/create-composite`, detail items dikirim sebagai **object** dengan tiga property: `insert`, `update`, dan `delete` sehingga memungkinkan ketiga operasi dalam satu request.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/stock-inbound/update-composite
```

---

## Format Request

### Struktur

```json
{
  "<nama_tabel_header>": {
    "<primary_key>": "existing-uuid",
    "field_to_update": "new-value",
    "<nama_tabel_detail>": {
      "delete": [{ "<detail_pk>": "uuid-to-delete" }],
      "update": [{ "<detail_pk>": "uuid-to-update", "field": "new-value" }],
      "insert": [{ "line_number": 3, "field": "value" }]
    }
  }
}
```

- Primary key header **wajib** disertakan
- Detail items berupa **object** `{ insert: [], update: [], delete: [] }`, **bukan** array
- Setiap property (`insert`, `update`, `delete`) bersifat opsional
- Detail items opsional, boleh update header saja tanpa menyertakan detail

### Contoh Request — Update Header Saja

```json
{
  "stock_inbound": {
    "stock_inbound_id": "d4e57872-17df-450e-b3bd-c0fac1dcb485",
    "status": "confirmed",
    "notes": "Sudah diterima dan dicek"
  }
}
```

### Contoh Request — Full Update dengan Detail Operations

```json
{
  "stock_inbound": {
    "stock_inbound_id": "d4e57872-17df-450e-b3bd-c0fac1dcb485",
    "notes": "Revisi lengkap detail items",
    "stock_inbound_item": {
      "delete": [
        { "stock_inbound_item_id": "e5f6g7h8-..." }
      ],
      "update": [
        {
          "stock_inbound_item_id": "a1b2c3d4-...",
          "qty_received": 15,
          "unit_price": 120000
        }
      ],
      "insert": [
        {
          "line_number": 3,
          "item_product_id": "c9d0e1f2-...",
          "qty_received": 5,
          "uom": "box",
          "unit_price": 250000
        }
      ]
    }
  }
}
```

---

## Detail Satu Baris (`one`)

RDF yang memakai [`masterDetail.details[]`](../catalogs/rdf/master-detail.md#beberapa-detail-dalam-satu-rdf-details) dapat memuat detail `one`, yaitu detail dengan paling banyak satu baris per header. Berbeda dengan detail `many`, detail `one` dikirim sebagai objek atau `null` di key `detailId`-nya, bukan sebagai struktur `{insert, update, delete}`.

```json
{
  "business_partner": {
    "business_partner_id": "d4e57872-17df-450e-b3bd-c0fac1dcb485",
    "partner_name": "PT Komponen Prima Abadi",
    "vendor": { "vendor_id": "a1b2c3d4-...", "payment_term_days": 45 },
    "customer": { "customer_code": "CUST-004", "credit_limit": 5000000 }
  }
}
```

Nilai key detail `one` menentukan operasinya.

| Nilai key detail `one` | Perilaku |
|------------------------|----------|
| Key tidak dikirim | Detail tidak disentuh |
| Objek tanpa primary key detail | Insert. Unique pada foreign key menolak insert kedua untuk header yang sama dengan `409` `UNIQUE_VIOLATION` |
| Objek dengan primary key detail | Update baris detail milik header ini. Primary key milik header lain dilewati tanpa error dan tanpa perubahan |
| `null` | Delete baris detail milik header ini. Ditolak dengan `400` `FOREIGN_KEY_VIOLATION` bila baris itu masih dirujuk data lain |

Nilai selain objek dan `null`, misalnya angka atau array, ditolak dengan `400`.

Pada contoh di atas, `vendor` memperbarui baris vendor yang sudah ada, sedangkan `customer` menambah baris customer baru. Response memuat seluruh detail `one` sebagai objek atau `null`, dan `_operations` dihitung per `detailId`.

```json
"_operations": {
  "vendor": { "deleted": 0, "updated": 1, "inserted": 0 },
  "customer": { "deleted": 0, "updated": 0, "inserted": 1 }
}
```

### Penolakan dan Key Error

Dua detail `one` dapat punya kolom bernama sama, sehingga pelanggaran constraint pada detail `one` memakai key `{detailId}.{kolom}` di map `details`. Field header dan detail `many` tetap memakai format error yang sudah ada.

| HTTP Status | Kondisi | Key di `details` |
|:-----------:|---------|------------------|
| 400 | Nilai key detail `one` bukan objek dan bukan `null`, dengan pesan `Property "customer" must be an object or null` | Tanpa `details` |
| 400 | `null` pada detail yang masih dirujuk data lain (`FOREIGN_KEY_VIOLATION`), dengan pesan `Data is still referenced by other records` | `vendor.vendor_id` |
| 400 | CHECK atau NOT NULL pada kolom detail (`CHECK_VIOLATION` atau `NOT_NULL_VIOLATION`) | `customer.payment_term_days` |
| 409 | Insert detail `one` kedua, atau unique constraint pada kolom detail (`UNIQUE_VIOLATION`) | `customer.business_partner_id` |

Seluruh operasi berjalan dalam satu transaction, sehingga penolakan di salah satu detail membatalkan perubahan header dan detail lain.

---

## Body Options (Opsional)

Endpoint `/update-composite` mendukung format request body alternatif dengan wrapper `{data, options}`. Format ini memungkinkan pengiriman opsi tambahan yang dapat dibaca oleh component engine handler dan processor.

### Format dengan Options

```json
{
  "data": {
    "stock_inbound": {
      "stock_inbound_id": "d4e57872-17df-450e-b3bd-c0fac1dcb485",
      "notes": "Revisi detail items",
      "stock_inbound_item": {
        "update": [
          {
            "stock_inbound_item_id": "a1b2c3d4-...",
            "qty_received": 15
          }
        ],
        "insert": [
          {
            "line_number": 3,
            "item_product_id": "c9d0e1f2-...",
            "qty_received": 5,
            "uom": "box",
            "unit_price": 250000
          }
        ],
        "delete": []
      }
    }
  },
  "options": {
    "recalculate_totals": true,
    "notify_warehouse": true
  }
}
```

### Aturan

- Body harus memiliki **tepat 2 key**: `data` dan `options`
- `data` harus berupa object (berisi root key tabel header + detail operations)
- `options` harus berupa object (berisi flag/parameter tambahan)
- Setelah `data` diekstrak, struktur di dalamnya diproses sama seperti format composite standar (root key, header fields, detail `{insert, update, delete}`)
- Jika format ini tidak terdeteksi, body diproses sebagai format standar (backward compatible)
- Nilai `options` tersedia di component handler sebagai template variable `{options}` dan di processor melalui `req.bodyOptions`

### `options.expectedVersion`

Berlaku hanya bila RDF endpoint ini mendeklarasikan blok [`concurrency`](../catalogs/rdf/concurrency.md). Sama seperti pada `/update` ([detail lengkap](./endpoint-update.md#optionsexpectedversion)), tetapi pemeriksaan dan kenaikan versi hanya berlaku pada baris **master**; detail items tidak punya versi sendiri. Berlaku juga pada request yang **hanya** mengubah detail (`insert`/`update`/`delete` tanpa field header berubah): selama `concurrency` dideklarasikan, versi master tetap dinaikkan meski tidak ada field header di body. Syarat `WHERE ... AND versionColumn = ?` sendiri hanya aktif bila `options.expectedVersion` dikirim, persis seperti `/update`.

```json
{
  "data": {
    "stock_inbound": {
      "stock_inbound_id": "d4e57872-17df-450e-b3bd-c0fac1dcb485",
      "stock_inbound_item": {
        "update": [{ "stock_inbound_item_id": "a1b2c3d4-...", "qty_received": 15 }]
      }
    }
  },
  "options": {
    "expectedVersion": 1
  }
}
```

---

## Format Response

### Response Sukses

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "message": "stock-inbound data successfully updated (with detail items)",
  "data": {
    "stock_inbound_id": "d4e57872-...",
    "inbound_number": "INB/2026/001",
    "notes": "Revisi lengkap detail items",
    "total_items": 2,
    "total_qty": 20,
    "total_amount": 3050000,
    "stock_inbound_item": [
      { "stock_inbound_item_id": "a1b2c3d4-...", "qty_received": 15, "unit_price": 120000, "total_amount": 1800000 },
      { "stock_inbound_item_id": "f3g4h5i6-...", "qty_received": 5, "unit_price": 250000, "total_amount": 1250000 }
    ],
    "_operations": {
      "deleted": 1,
      "updated": 1,
      "inserted": 1
    }
  },
  "timestamp": "2026-03-30T10:00:00.000Z"
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `data` | object | Header setelah diperbarui, berisi field yang terdaftar di `fieldName` |
| `data.stock_inbound_item` | array | **Seluruh** detail items yang tersisa (bukan hanya yang diubah), urut menurut kolom pengurut hasil resolusi detail. Setiap item berisi field di `detailConfig.fieldName` ditambah primary key detail, foreign key, dan kolom `autoCalculateFields` |
| `data._operations` | object | Ringkasan operasi yang dilakukan |

Pada RDF ber-`concurrency`, `data` memuat nilai `versionColumn` master yang terbaru (dinaikkan 1 untuk mode `version`). **`noChanges` tidak berlaku pada endpoint ini**, berbeda dengan `/update` ([detail](./endpoint-update.md#format-response)). `/update-composite` selalu menjalankan UPDATE header bila `concurrency` dideklarasikan, karena kombinasi field header dan operasi detail yang bisa saja tanpa perubahan header sekalipun membuat deteksi "tanpa perubahan" ambigu untuk composite. Response sukses endpoint ini tidak pernah memuat field `noChanges`.

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
| 400 | Root key tidak sesuai | `Payload must have property "stock_inbound"` |
| 400 | Primary key tidak ada | `Primary key (stock_inbound_id) is required for update` |
| 400 | Detail dikirim sejajar header, disertai `example` bentuk body | `Property "stock_inbound_item" is missing inside "stock_inbound". Found "stock_inbound_item" at top level; move it inside "stock_inbound"` |
| 400 | Detail format salah (array bukan object) | `Property "stock_inbound_item" must be an object with structure {insert: [], update: [], delete: []}` |
| 400 | Validasi field header gagal (`code: VALIDATION_FAILED`) | `{ details: { field: ["message"] }, errors: ["message"] }` |
| 400 | Token versi tidak valid (RDF ber-`concurrency`) | `options.expectedVersion must be an integer` |
| 404 | Header tidak ditemukan | `stock-inbound data not found` |
| 409 | Versi baris master berbeda (RDF ber-`concurrency`) | `This record was modified by another user after you opened it.` |
| 409 | Unique constraint violation | `A record with this value already exists` |
| 400 | Ditolak handler component `onBeforeCompositeUpdate` atau `onAfterCompositeUpdate`. Data yang ditulis handler lewat `services.db` ikut dibatalkan, kecuali pada SQLite. Penolakan di `onAfterCommitCompositeUpdate` tidak menghasilkan error | Pesan dari handler saja, tanpa nama event, nama fungsi, atau path file handler |
| 400 | Foreign key constraint (`code: FOREIGN_KEY_VIOLATION`) | `Referenced data not found`, dengan `details` per kolom bila kolom header diketahui |
| 400 | CHECK atau NOT NULL ditolak database (`code: CHECK_VIOLATION` / `NOT_NULL_VIOLATION`) | `Invalid data`, dengan `details` per field header |
| 500 | Error database | `An error occurred while updating stock-inbound data` |

> Pelanggaran constraint pada tabel detail `many` dibalas tanpa `details`, sedangkan detail `one` memakai key `{detailId}.{kolom}` (lihat [Detail Satu Baris](#detail-satu-baris-one)). Rincian bentuk `400` ada di
> [`catalogs/rdf/field-validation.md`](../catalogs/rdf/field-validation.md#format-error-response-400-pelanggaran-constraint-database).
>
> **Field `details` pada 409:** response unique violation (`code: UNIQUE_VIOLATION`) menyertakan map `details` =
> `{ "<kolom>": ["<pesan>"] }` yang menunjuk kolom pelanggar (untuk constraint multi-kolom,
> tiap kolom dipetakan ke `"A record with this combination of values already exists"`).
> Field ini selalu muncul bila kolom pelanggar dapat diekstrak; bila tidak, field ini tidak disertakan. Detail:
> [`README` — Duplicate entry](./README.md#409--conflict).
>
> **409 versi baris berbeda** memakai bentuk `details` yang berbeda:
> `{ "currentVersion": 2, "updated_by": "system", "updated_at": "..." }`, sama persis dengan
> `/update` ([detail lengkap](./endpoint-update.md#409--conflict-versi-baris-berbeda)). Field ini
> selalu ada pada response konflik versi; `updated_by`/`updated_at` bernilai `null` bila kolom
> audit tidak ada di RDF.

---

## Urutan Eksekusi

Dalam satu transaction, operasi detail dijalankan dalam urutan tetap:

```
1. DELETE  → Hapus detail items berdasarkan PK
2. UPDATE  → Perbarui detail items existing
3. INSERT  → Tambah detail items baru (auto UUID PK, auto FK)
4. RECALCULATE → Hitung ulang header totals dari semua detail yang tersisa
5. UPDATE HEADER → Simpan field header + totals baru (RDF ber-`concurrency`: syarat versi di WHERE + kenaikan versi terjadi di langkah ini)
```

Pada RDF ber-[`concurrency`](../catalogs/rdf/concurrency.md), langkah UPDATE HEADER tetap berjalan meski request hanya mengubah detail (tidak ada field header yang dikirim), khusus untuk menjaga kenaikan versi dan pemeriksaan token tetap berlaku pada request detail-only.

## Kunci Dokumen

Berlaku hanya bila RDF endpoint ini mendeklarasikan blok [`documentLock`](../catalogs/rdf/document-lock.md). Header dokumen dikunci setelah transaksi dibuka, sebelum operasi detail dijalankan.

### 409 — Document Locked

Response memakai HTTP status `409 Conflict`. Error ini terjadi ketika dokumen sudah final atau sudah dirujuk dokumen lain yang statusnya bukan void. Operasi ditolak sebelum hook `onBefore*` dipanggil, dan tidak ada data yang berubah.

```json
{
  "success": false,
  "error": "Document locked",
  "message": "Cannot update: document is locked. Used by GR-2026-0003",
  "details": {
    "is_lock": true,
    "lock_reason": "Used by GR-2026-0003"
  },
  "timestamp": "2026-10-25T09:15:00.000Z"
}
```

`details.lock_reason` berisi `Used by <nomor>` bila dokumen dirujuk, atau label status final seperti `Received` bila statusnya sudah final. Kata kerja di awal `message` mengikuti action, yaitu `update` pada endpoint ini.

Dokumen terkunci menolak seluruh request, termasuk request yang hanya berisi operasi detail `insert`, `update`, atau `delete`.

### Kenaikan Versi

Setiap penulisan yang berhasil menaikkan `versionField` header sebanyak 1. Pada RDF yang juga memakai [`concurrency`](../catalogs/rdf/concurrency.md), versi tetap naik 1 per penulisan, bukan 2.

### Sisi Perujuk

Bila RDF endpoint ini juga mendeklarasikan `documentLockTargets`, `update-composite` memeriksa versi dokumen yang dirujuk lewat `options.documentVersions`. Format request, aturan, dan response error `400` serta `409` yang terkait dijelaskan di [`/create`](./endpoint-create.md#versi-dokumen-yang-dirujuk-optionsdocumentversions).
Pemeriksaan berlaku untuk foreign key di header dan di item detail yang baru atau berubah. Untuk foreign key yang menunjuk baris dokumen, kunci pada `options.documentVersions` tetap berupa id header di tabel yang dikunci.

```json
{
  "data": {
    "vendor_bill": {
      "vendor_bill_id": "vb_0002",
      "vendor_bill_item": {
        "insert": [
          {
            "purchase_order_item_id": "poi_0013",
            "quantity": 5
          }
        ]
      }
    }
  },
  "options": {
    "documentVersions": {
      "purchase_order": {
        "po_0007": 3
      }
    }
  }
}
```

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| CRUD Composite (Deep Dive) | — | Dokumentasi lengkap master-detail |
| Field Validation | — | Validasi field header dan detail |
| Cache | — | Invalidasi cache setelah update berhasil |
| Body Options | — | Opsi tambahan via format `{data, options}` |
| Concurrency | [`../catalogs/rdf/concurrency.md`](../catalogs/rdf/concurrency.md) | Optimistic concurrency control pada baris master (`options.expectedVersion`, 409 Conflict) |
| Kunci Dokumen | [`../catalogs/rdf/document-lock.md`](../catalogs/rdf/document-lock.md) | Dokumen terkunci ditolak `409 Document locked`, dan dokumen perujuk memeriksa versi dokumen yang dirujuk |
