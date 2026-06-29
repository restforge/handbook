# Read Composite — Endpoint Pengambilan Data Master-Detail

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/read-composite` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `200 OK` |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Parameter Wajib** | `where` (kondisi filter header) |
| **Hasil** | Header record(s) + detail items per header |
| **Cache** | **Tidak** di-cache |

---

## Ikhtisar

Endpoint `/read-composite` mengambil data header beserta seluruh detail items yang terkait dalam satu response. Setiap header record secara otomatis dilengkapi dengan array detail items yang di-query menggunakan `detailQuery` dari konfigurasi payload.

Detail query dapat menyertakan kolom dari tabel relasi (JOIN), misalnya `product_code` dan `product_name` dari tabel `item_product`.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/stock-inbound/read-composite
```

---

## Format Request

### Parameter

| Parameter | Tipe | Wajib | Keterangan |
|-----------|------|:-----:|------------|
| `where` | array | **Ya** | Kondisi filter header. Format: `[{ key, value }]` |

### Contoh Request

**1. Berdasarkan primary key:**
```json
{
  "where": [
    { "key": "stock_inbound_id", "value": "d4e57872-17df-450e-b3bd-c0fac1dcb485" }
  ]
}
```

**2. Berdasarkan status:**
```json
{
  "where": [
    { "key": "status", "value": "draft" }
  ]
}
```

**3. Multiple conditions (AND):**
```json
{
  "where": [
    { "key": "warehouse_id", "value": "e9287574-..." },
    { "key": "status", "value": "confirmed" }
  ]
}
```

---

## Format Response

### Response Sukses

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "count": 1,
  "data": [
    {
      "stock_inbound_id": "d4e57872-...",
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
          "product_code": "PRD-001",
          "product_name": "Keyboard Wireless",
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
          "product_code": "PRD-002",
          "product_name": "Mouse Optical",
          "qty_received": 20,
          "uom": "pcs",
          "unit_price": 50000,
          "total_amount": 1000000
        }
      ]
    }
  ],
  "timestamp": "2026-03-30T09:05:00.000Z"
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `count` | number | Jumlah header record yang cocok |
| `data` | array | Array header records, masing-masing berisi detail items |
| `data[].{detail_table}` | array | Detail items per header (di-query via `detailQuery`) |

> **Catatan:** Kolom seperti `product_code` dan `product_name` berasal dari JOIN di `detailQuery`, bukan dari tabel detail langsung.

### Response Error

| HTTP Status | Kondisi | Contoh Message |
|:-----------:|---------|----------------|
| 400 | Payload kosong | `Payload cannot be empty` |
| 400 | WHERE tidak ada | `Property where is required` |
| 500 | Error database | `An error occurred while reading stock-inbound data` |

---

## Konfigurasi Detail Query

Detail items di-query menggunakan `detailConfig.detailQuery` dari payload:

```json
{
  "masterDetail": {
    "detailConfig": {
      "detailQuery": "SELECT a.*, b.product_code, b.product_name FROM stock_inbound_item a LEFT JOIN item_product b ON a.item_product_id = b.item_product_id WHERE a.stock_inbound_id = $1 ORDER BY a.line_number"
    }
  }
}
```

Parameter `$1` secara otomatis diisi dengan primary key header dari setiap record.

---

## Perbedaan dengan /first dan /read

| Aspek | `/read-composite` | `/first` | `/read` |
|-------|-------------------|---------|--------|
| Detail items | Ya (nested array) | Tidak | Tidak |
| Hasil | Array header + detail | 1 record | Array flat |
| Cache | Tidak | Tidak | Ya |
| JOIN di detail | Via `detailQuery` | — | — |

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| CRUD Composite (Deep Dive) | — | Dokumentasi lengkap master-detail |
