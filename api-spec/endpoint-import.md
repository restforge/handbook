# Import — Endpoint Impor Data dari Excel

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Langkah 1** | `POST /import-upload` — Upload dan parse file Excel |
| **Langkah 2** | `POST /import-preview` — Trigger validasi dan preview (async) |
| **Langkah 3** | `POST /import-commit` — Trigger eksekusi insert/update ke database (async) |
| **Polling** | `GET /import-status?job_id={id}` — Cek status job preview maupun commit |
| **Format Input** | Excel (.xlsx) |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Proses** | Asinkron (async job dengan polling status) |
| **Upsert Strategy** | `update_existing`, `insert_only`, `skip_existing` |

---

## Ikhtisar

Fitur import memungkinkan data dari file Excel (.xlsx) dimasukkan ke database secara batch. Alur utama terdiri dari 3 langkah (upload, preview, commit) yang berjalan asinkron. Endpoint `/import-status` digunakan untuk polling progress di antara langkah-langkah tersebut, baik setelah preview maupun setelah commit. Fitur ini mendukung auto-generate UUID, lookup field resolution (nama → ID), dan konfigurasi upsert strategy.

---

## Langkah 1 — Upload File (POST /import-upload)

### Format Request

**Content-Type:** `multipart/form-data`

| Field | Tipe | Wajib | Keterangan |
|-------|------|:-----:|------------|
| `file` | File | **Ya** | File Excel (.xlsx) |
| `upsertMode` | string | Tidak | Override strategy: `update_existing`, `insert_only`, `skip_existing` |
| `upsertKeys` | string | Tidak | JSON string, override kolom upsert key |

### Response Sukses

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "data": {
    "upload_id": "x1y2z3a4b5c6",
    "total_rows": 150,
    "preview": [
      { "supplier_code": "SUP-001", "supplier_name": "PT Maju Jaya", "city": "Jakarta" },
      { "supplier_code": "SUP-002", "supplier_name": "CV Sentosa", "city": "Surabaya" }
    ]
  }
}
```

---

## Langkah 2 — Preview dan Validasi (POST /import-preview)

### Format Request

```json
{
  "upload_id": "x1y2z3a4b5c6"
}
```

### Response Sukses

```json
{
  "success": true,
  "data": {
    "job_id": "p1q2r3s4t5u6",
    "status": "validating"
  }
}
```

---

## Polling Status (GET /import-status)

Endpoint ini digunakan setelah Langkah 2 (untuk cek hasil preview/validasi) dan setelah Langkah 3 (untuk cek hasil commit). Karena proses berjalan asinkron, client harus melakukan polling sampai status berubah menjadi `preview_ok`/`preview_error` atau `completed`/`failed`.

### Format Request

```
GET /api/{project}/{endpoint}/import-status?job_id=p1q2r3s4t5u6
```

### Response — Status `preview_ok`

```json
{
  "success": true,
  "data": {
    "job_id": "p1q2r3s4t5u6",
    "status": "preview_ok",
    "progress": 100,
    "preview": {
      "total_rows": 150,
      "insert_count": 100,
      "update_count": 40,
      "skip_count": 0,
      "error_count": 0
    }
  }
}
```

### Response — Status `preview_error`

```json
{
  "success": true,
  "data": {
    "job_id": "p1q2r3s4t5u6",
    "status": "preview_error",
    "progress": 100,
    "preview": {
      "total_rows": 150,
      "insert_count": 95,
      "update_count": 40,
      "error_count": 5,
      "errors": [
        {
          "rowIndex": 3,
          "rowData": { "product_code": "", "product_name": "Widget A" },
          "errors": ["Required field 'product_code' is empty"]
        }
      ]
    }
  }
}
```

### Response — Status `completed` (Setelah Commit)

```json
{
  "success": true,
  "data": {
    "job_id": "c1d2e3f4g5h6",
    "status": "completed",
    "progress": 100,
    "result": {
      "total_rows": 150,
      "inserted_rows": 100,
      "updated_rows": 40,
      "skipped_rows": 0,
      "failed_rows": 0,
      "errors": []
    }
  }
}
```

---

## Langkah 3 — Commit ke Database (POST /import-commit)

### Format Request

```json
{
  "preview_job_id": "p1q2r3s4t5u6"
}
```

> **Prasyarat:** Hanya dapat dipanggil jika status preview adalah `preview_ok`. Jika status `preview_error`, commit ditolak.

### Response Sukses

```json
{
  "success": true,
  "data": {
    "job_id": "c1d2e3f4g5h6",
    "status": "processing"
  }
}
```

Gunakan `GET /import-status?job_id=c1d2e3f4g5h6` untuk memantau progres commit.

---

## Alur Proses

```
1. UPLOAD → Parse Excel, validasi struktur, simpan di Redis (TTL 1 jam)
                ↓
2. PREVIEW → Auto-generate UUID, lookup field resolution, validasi data
                ↓
   ├── preview_ok    → Lanjut ke COMMIT
   └── preview_error → Perbaiki Excel, upload ulang
                ↓
3. COMMIT → Batch INSERT/UPDATE (per 100 baris), atomic transaction per batch
                ↓
   └── completed / failed
```

---

## Upsert Strategy

| Strategy | Row Sudah Ada | Row Baru |
|----------|:------------:|:--------:|
| `update_existing` | UPDATE | INSERT |
| `insert_only` | SKIP + error | INSERT |
| `skip_existing` | SKIP (silent) | INSERT |

Deteksi duplikat berdasarkan kolom `upsertKeys` di `importConfig`.

---

## Konfigurasi Payload

```json
{
  "action": { "import": true },
  "importConfig": {
    "enabled": true,
    "upsertKeys": ["supplier_id"],
    "upsertStrategy": "update_existing",
    "requiredFields": ["supplier_code", "supplier_name"],
    "maxFileSize": "10MB",
    "allowedFormats": ["xlsx"],
    "chunkSize": 100,
    "lookupFields": {
      "category_name": {
        "targetField": "category_id",
        "lookupTable": "category",
        "lookupColumn": "category_name",
        "lookupIdColumn": "category_id",
        "required": true
      }
    }
  }
}
```

| Property | Tipe | Keterangan |
|----------|------|------------|
| `upsertKeys` | array | Kolom untuk deteksi duplikat |
| `upsertStrategy` | string | Strategy upsert default |
| `requiredFields` | array | Kolom wajib di Excel |
| `chunkSize` | number | Ukuran batch INSERT/UPDATE (default: 100) |
| `lookupFields` | object | Konfigurasi resolusi foreign key (nama → ID) |

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| Export Import (Deep Dive) | — | Dokumentasi lengkap export dan import |
| Import Auto-UUID (upsertKeys) | — | Auto-generate UUIDv7 untuk field upsertKeys saat import |
