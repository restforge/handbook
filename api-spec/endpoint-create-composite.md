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
    "created_at": "2026-03-30T09:00:00.000Z",
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

### Response Error

| HTTP Status | Kondisi | Contoh Message |
|:-----------:|---------|----------------|
| 400 | Root key tidak sesuai nama tabel | `Payload must have property "stock_inbound"` |
| 400 | Detail array kosong | `Property "stock_inbound_item" must be a non-empty array` |
| 400 | Validasi field gagal | `{ errors: { field: ["message"] } }` |
| 409 | Unique constraint violation | `A record with this value already exists` |
| 400 | Foreign key constraint | `Referenced data not found` |
| 500 | Error database / CHECK constraint | `An error occurred while creating stock-inbound data` |

> **Field `details` pada 409:** response unique violation menyertakan map `details` =
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
| `created_at` | Timestamp saat ini | Jika ada di fieldName |
| `updated_at` | Timestamp saat ini | Jika ada di fieldName |

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
