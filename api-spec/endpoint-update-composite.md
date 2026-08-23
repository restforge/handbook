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
    "updated_at": "2026-03-30T10:00:00.000Z",
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
| `data.stock_inbound_item` | array | **Seluruh** detail items yang tersisa (bukan hanya yang diubah) |
| `data._operations` | object | Ringkasan operasi yang dilakukan |

### Response Error

| HTTP Status | Kondisi | Contoh Message |
|:-----------:|---------|----------------|
| 400 | Root key tidak sesuai | `Payload must have property "stock_inbound"` |
| 400 | Primary key tidak ada | `Primary key (stock_inbound_id) is required for update` |
| 400 | Detail format salah (array bukan object) | `Property "stock_inbound_item" must be an object with structure {insert: [], update: [], delete: []}` |
| 400 | Validasi field gagal | `{ errors: { field: ["message"] } }` |
| 404 | Header tidak ditemukan | `stock-inbound data not found` |
| 409 | Unique constraint violation | `A record with this value already exists` |
| 400 | Foreign key constraint | `Referenced data not found` |
| 500 | Error database | `An error occurred while updating stock-inbound data` |

> **Field `details` pada 409:** response unique violation menyertakan map `details` =
> `{ "<kolom>": ["<pesan>"] }` yang menunjuk kolom pelanggar (untuk constraint multi-kolom,
> tiap kolom dipetakan ke `"A record with this combination of values already exists"`).
> Field ini selalu muncul bila kolom pelanggar dapat diekstrak; bila tidak, field ini tidak disertakan. Detail:
> [`README` — Duplicate entry](./README.md#409--conflict).

---

## Urutan Eksekusi

Dalam satu transaction, operasi detail dijalankan dalam urutan tetap:

```
1. DELETE  → Hapus detail items berdasarkan PK
2. UPDATE  → Perbarui detail items existing
3. INSERT  → Tambah detail items baru (auto UUID PK, auto FK)
4. RECALCULATE → Hitung ulang header totals dari semua detail yang tersisa
5. UPDATE HEADER → Simpan field header + totals baru
```

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| CRUD Composite (Deep Dive) | — | Dokumentasi lengkap master-detail |
| Field Validation | — | Validasi field header dan detail |
| Cache | — | Invalidasi cache setelah update berhasil |
| Body Options | — | Opsi tambahan via format `{data, options}` |
