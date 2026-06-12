# Export — Endpoint Ekspor Data ke Excel

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Tahap 1** | `POST /api/{project}/{endpoint}/export` — Trigger export job |
| **Tahap 2** | `GET /api/{project}/{endpoint}/export-status?job_id={id}` — Cek status |
| **Tahap 3** | `GET /api/{project}/{endpoint}/export-download?job_id={id}` — Download file |
| **Format Output** | Excel (.xlsx) |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Proses** | Asinkron (async job) |
| **Filter** | WHERE clause dan sort_columns didukung |
| **Konfigurasi** | `action.export: true` dan `exportQuery` di payload |

---

## Ikhtisar

Fitur export memungkinkan data diekspor ke file Excel (.xlsx) secara asinkron. Proses terdiri dari 3 endpoint terpisah: trigger job, polling status, dan download file hasil. Pendekatan asinkron mencegah timeout pada dataset besar.

---

## Tahap 1 — Trigger Export (POST /export)

### Format Request

Seluruh parameter opsional. Body kosong `{}` mengekspor seluruh data.

| Parameter | Tipe | Wajib | Keterangan |
|-----------|------|:-----:|------------|
| `format` | string | Tidak | Format output. Saat ini hanya `"xlsx"` |
| `where` | object | Tidak | Filter data. Lihat [Format WHERE](README.md#format-where-clause) |
| `sort_columns` | array | Tidak | Pengurutan data. Lihat [Format Sort Columns](README.md#format-sort-columns) |

**Contoh — Export dengan filter:**
```json
{
  "where": {
    "logic": "AND",
    "conditions": [
      { "key": "is_active", "operator": "=", "value": true },
      { "key": "stock", "operator": ">", "value": 0 }
    ]
  },
  "sort_columns": [
    { "column": "product_name", "direction": "ASC" }
  ]
}
```

### Response Sukses

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "data": {
    "job_id": "a1b2c3d4e5f6",
    "status": "pending"
  }
}
```

---

## Tahap 2 — Cek Status (GET /export-status)

### Format Request

```
GET /api/{project}/{endpoint}/export-status?job_id=a1b2c3d4e5f6
```

### Response — Status `pending`

```json
{
  "success": true,
  "data": {
    "job_id": "a1b2c3d4e5f6",
    "status": "pending",
    "progress": 0,
    "created_at": "2026-03-30T10:30:00Z"
  }
}
```

### Response — Status `processing`

```json
{
  "success": true,
  "data": {
    "job_id": "a1b2c3d4e5f6",
    "status": "processing",
    "progress": 45,
    "total_rows": 500,
    "processed_rows": 225
  }
}
```

### Response — Status `completed`

```json
{
  "success": true,
  "data": {
    "job_id": "a1b2c3d4e5f6",
    "status": "completed",
    "progress": 100,
    "total_rows": 500,
    "processed_rows": 500,
    "file_url": "/api/mini-inventory/supplier/export-download?job_id=a1b2c3d4e5f6",
    "created_at": "2026-03-30T10:30:00Z",
    "completed_at": "2026-03-30T10:30:05Z"
  }
}
```

### Response — Status `failed`

```json
{
  "success": true,
  "data": {
    "job_id": "a1b2c3d4e5f6",
    "status": "failed",
    "progress": 25,
    "error": "Database connection timeout"
  }
}
```

**Alur status:** `pending` → `processing` → `completed` / `failed`

---

## Tahap 3 — Download File (GET /export-download)

### Format Request

```
GET /api/{project}/{endpoint}/export-download?job_id=a1b2c3d4e5f6
```

### Response

Binary file Excel (.xlsx) dengan header:
```
Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
Content-Disposition: attachment; filename="supplier_export_2026-03-30.xlsx"
```

---

## Konfigurasi Payload

```json
{
  "action": { "export": true },
  "exportQuery": "SELECT supplier_id, supplier_code, supplier_name, city FROM supplier",
  "columnFormats": {
    "purchase_price": { "type": "number", "format": "#,##0.00" },
    "stock": { "type": "number", "format": "#,##0" },
    "is_active": { "type": "boolean" },
    "created_date": { "type": "date" }
  }
}
```

`exportQuery` mendukung referensi file: `"file:query/supplier_export.sql"`.

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| Export Import (Deep Dive) | — | Dokumentasi lengkap export dan import |
| Sort Column | — | Pengurutan data saat export |
