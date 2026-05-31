# Spesifikasi API Endpoint — RESTForge

> Index dan referensi pola bersama untuk seluruh endpoint REST API yang di-generate RESTForge: pola URL, envelope response, kode status HTTP, format WHERE clause, dan sort columns.

---

## Daftar Endpoint (Endpoint Index)

### CRUD Standar (Standard CRUD)

| # | Endpoint | Metode | URL | Deskripsi |
|---|----------|--------|-----|-----------|
| 1 | [create](endpoint-create.md) | POST | `/api/{project}/{endpoint}/create` | Membuat data baru |
| 2 | [read](endpoint-read.md) | POST | `/api/{project}/{endpoint}/read` | Mengambil data (list/paginasi) |
| 3 | [first](endpoint-first.md) | POST | `/api/{project}/{endpoint}/first` | Mengambil 1 record (View/Edit form) |
| 4 | [update](endpoint-update.md) | POST | `/api/{project}/{endpoint}/update` | Memperbarui data existing |
| 5 | [delete](endpoint-delete.md) | POST | `/api/{project}/{endpoint}/delete` | Menghapus data |
| 6 | [adjust](endpoint-adjust.md) | POST | `/api/{project}/{endpoint}/adjust` | Penyesuaian atomik field numerik |

### Workflow dan Status (Workflow & Status)

| # | Endpoint | Metode | URL | Deskripsi |
|---|----------|--------|-----|-----------|
| 7 | [change-status](endpoint-change-status.md) | POST | `/api/{project}/{endpoint}/change-status` | Mengubah status record dengan validasi transisi + hook API |

### Query dan Retrieval (Query & Retrieval)

| # | Endpoint | Metode | URL | Deskripsi |
|---|----------|--------|-----|-----------|
| 8 | [datatables](endpoint-datatables.md) | POST | `/api/{project}/{endpoint}/datatables` | Server-side processing DataTables |
| 9 | [lookup](endpoint-lookup.md) | GET / POST | `/api/{project}/{endpoint}/lookup` | Data untuk dropdown/autocomplete |
| 10 | [aggregate](endpoint-aggregate.md) | POST | `/api/{project}/{endpoint}/aggregate` | Agregasi data (COUNT, SUM, AVG, MIN, MAX) |

### CRUD Composite — Master-Detail

| # | Endpoint | Metode | URL | Deskripsi |
|---|----------|--------|-----|-----------|
| 11 | [create-composite](endpoint-create-composite.md) | POST | `/api/{project}/{endpoint}/create-composite` | Membuat data header + detail items |
| 12 | [update-composite](endpoint-update-composite.md) | POST | `/api/{project}/{endpoint}/update-composite` | Memperbarui header + insert/update/delete detail |
| 13 | [read-composite](endpoint-read-composite.md) | POST | `/api/{project}/{endpoint}/read-composite` | Mengambil data header + detail items |

### Export dan Import (Export & Import)

| # | Endpoint | Metode | URL | Deskripsi |
|---|----------|--------|-----|-----------|
| 14 | [export](endpoint-export.md) | POST + GET | `/api/{project}/{endpoint}/export` | Ekspor data ke Excel (.xlsx) |
| 15 | [import](endpoint-import.md) | POST + GET | `/api/{project}/{endpoint}/import-*` | Impor data dari Excel (.xlsx) |

---

## Pola URL (URL Pattern)

Seluruh endpoint mengikuti pola URL yang konsisten:

```
{METHOD} /api/{project}/{endpoint}/{action}
```

| Segmen | Keterangan | Contoh |
|--------|------------|--------|
| `{project}` | Nama project yang didaftarkan saat generate (`--project`) | `mini-inventory` |
| `{endpoint}` | Nama endpoint (resource), sesuai nama file yang di-generate | `supplier`, `item-product` |
| `{action}` | Aksi yang dilakukan | `create`, `read`, `first`, `update`, `delete`, `adjust`, `change-status`, `datatables`, `lookup`, `aggregate`, `create-composite`, `export`, dll. |

**Contoh URL lengkap:**
```
POST http://localhost:3000/api/mini-inventory/supplier/create
POST http://localhost:3000/api/mini-inventory/item-product/read
GET  http://localhost:3000/api/mini-inventory/supplier/lookup?search=abc
```

---

## Format Response Umum (Common Response Format)

### Response Sukses (Success Response)

Seluruh endpoint (kecuali `/datatables`) menggunakan envelope response berikut:

```json
{
  "success": true,
  "message": "Deskripsi operasi yang berhasil",
  "data": {},
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `success` | boolean | Selalu `true` untuk operasi yang berhasil |
| `message` | string | Pesan deskriptif hasil operasi |
| `data` | object / array | Hasil data (format bervariasi per endpoint) |
| `timestamp` | string | Waktu eksekusi dalam format ISO 8601 |

> **Catatan:** Endpoint `/datatables` menggunakan format response khusus DataTables (`{ draw, recordsTotal, recordsFiltered, data }`) tanpa envelope `success`. Lihat [endpoint-datatables.md](endpoint-datatables.md).

### Response Error (Error Response)

```json
{
  "success": false,
  "error": "Kategori Error",
  "message": "Pesan error yang dapat dibaca manusia",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `success` | boolean | Selalu `false` untuk operasi yang gagal |
| `error` | string | Kategori error (e.g., `"Invalid payload"`, `"Validation failed"`) |
| `message` | string | Pesan error detail |
| `details` | string | Detail error teknis. **Hanya muncul di development mode** (`NODE_ENV=development`) |
| `timestamp` | string | Waktu eksekusi dalam format ISO 8601 |

### Response Validasi (Validation Error Response)

Endpoint `/create` dan `/update` yang memiliki `fieldValidation` menghasilkan format error khusus:

```json
{
  "success": false,
  "error": "Validation failed",
  "message": "Invalid data",
  "errors": {
    "supplier_code": ["Field supplier_code is required"],
    "email": ["Field email must be a valid email address"],
    "price": ["Field price must be at least 0"]
  },
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `errors` | object | Key = nama field, Value = array pesan error per field |

Daftar lengkap constraint dan pesan error tersedia pada [katalog Field Validation](../commands/restforge-backend/catalog/field-validation.md).

---

## Definisi Kode Status HTTP (HTTP Status Code Definitions)

Setiap HTTP status code memiliki scope dan definisi yang jelas, ditentukan berdasarkan jenis error yang terjadi saat request diproses.

---

### 201 — Created

**Definisi:** Operasi pembuatan data berhasil.

**Scope:** Hanya digunakan oleh endpoint yang membuat record baru.

| Endpoint | Kondisi |
|----------|---------|
| `/create` | INSERT berhasil |
| `/create-composite` | INSERT header + detail berhasil |

---

### 200 — OK

**Definisi:** Operasi berhasil (selain pembuatan data baru).

**Scope:** Semua endpoint selain `/create` dan `/create-composite`.

| Endpoint | Kondisi |
|----------|---------|
| `/read` | Query berhasil (termasuk jika data kosong) |
| `/first` | Query berhasil (termasuk jika `count: 0`) |
| `/update` | UPDATE berhasil |
| `/delete` | DELETE berhasil |
| `/adjust` | Adjustment berhasil |
| `/datatables` | Query berhasil |
| `/lookup` | Query berhasil |
| `/aggregate` | Agregasi berhasil |

---

### 400 — Bad Request

**Definisi:** Request tidak dapat diproses karena **kesalahan dari sisi client**, yaitu data yang dikirim tidak valid, tidak lengkap, atau tidak sesuai format.

**Prinsip:** Error ini terjadi **sebelum** operasi database utama dijalankan (kecuali FK constraint pada create/update).

**Scope:**

| Kategori | Error Field | Contoh Message |
|----------|-------------|----------------|
| **Payload kosong** | `Invalid payload` | `Payload cannot be empty` |
| **Field wajib hilang** | `Missing required field` | `Primary key (supplier_id) is required for update` |
| **Parameter WHERE hilang** | `Missing required field` | `Invalid request format: where parameter is required` |
| **Format WHERE salah** | `Invalid where format` | `Invalid where format` |
| **Format WHERE tidak didukung** | `Invalid where format` | `Advanced where format is not supported in /first endpoint` |
| **Field WHERE tidak valid** | `Invalid where field` | `Invalid field: unknown_column` |
| **Validasi field gagal** | `Validation failed` | `Invalid data` + object `errors` |
| **Field select tidak valid** | `Invalid select fields` | `Invalid field(s): unknown_column` |
| **Parameter pencarian terlalu panjang** | `Invalid parameter` | `Search value must not exceed 255 characters` |
| **Parameter paginasi tidak valid** | `Invalid parameter` | `Page must be greater than 0` |
| **Header HTTP hilang** | `Missing required header` | `x-correlation-id header is required` (jika Kafka enabled) |
| **Header X-Request-Mode salah** | `Invalid Request Mode` | `X-Request-Mode header must be set to dynamic` (endpoint `/lookup`) |
| **Adjustments array kosong** | `Invalid payload` | `adjustments array is required and must not be empty` |
| **Validasi adjustment** | `Validation error` | `Field "x" is not configured for adjustment` |
| **Validasi aggregate** | `Validation error` | `Invalid aggregate function "median"` |
| **FK constraint (create/update)** | `Foreign key constraint` | `Referenced data not found` (PostgreSQL error code `23503`) |

---

### 404 — Not Found

**Definisi:** Record yang dituju **tidak ditemukan** di database.

**Prinsip:** Hanya terjadi setelah query ke database yang **tidak menghasilkan data** ketika data **diharapkan ada** (operasi update/delete/adjust yang membutuhkan record existing).

**Scope:**

| Endpoint | Kondisi |
|----------|---------|
| `/update` | Record dengan primary key tersebut tidak ditemukan (0 rows ter-update) |
| `/delete` | Record tidak ada saat verifikasi sebelum delete |
| `/adjust` | Record tidak ditemukan sebelum adjustment |

**Yang BUKAN 404:**
- `/read` dengan data kosong → **200** (data kosong bukan error)
- `/first` dengan data kosong → **200** (`count: 0`, `data: []`)
- `/datatables` tanpa hasil → **200** (`recordsFiltered: 0`)

---

### 409 — Conflict

**Definisi:** Operasi gagal karena **konflik dengan state data saat ini** di database, yaitu constraint violation yang menunjukkan data sudah ada atau masih direferensikan.

**Scope:**

| Kategori | Error Field | Kondisi | PG Error Code |
|----------|-------------|---------|:-------------:|
| **Duplicate entry** | `Duplicate entry` | Unique constraint violation | `23505` |
| **FK constraint (delete)** | `Foreign key constraint` | Record masih direferensikan oleh tabel lain | `23503` |
| **Constraint violation (adjust)** | `Constraint violation` | Guard clause gagal (nilai akan di bawah minimum) | — |
| **Resource busy (change-status)** | `Resource busy` | Gagal acquire write lock (record sedang diproses request lain) | — |
| **Concurrent modification (change-status)** | `Concurrent modification` | Status berubah oleh request lain saat operasi berlangsung (guard clause) | — |

**Perbedaan FK constraint antara endpoint:**
- **Create/Update** → FK `23503` di-map ke **400** (data referensi tidak ditemukan = client error)
- **Delete** → FK `23503` di-map ke **409** (record tidak bisa dihapus karena masih digunakan = conflict)

**Catatan lock failure:** Pada `/update`, `/adjust`, `/delete`, kegagalan acquire lock di-map ke **500** (catch-all). Pada `/change-status`, kegagalan lock di-map khusus ke **409** `Resource busy`.

---

### 422 — Unprocessable Entity

**Definisi:** Request valid secara sintaksis, tetapi melanggar aturan transisi status (business rule) yang dikonfigurasi.

**Scope:** Hanya endpoint `/change-status` ketika `workflow.transitions` aktif.

| Kategori | Error Field | Contoh Message |
|----------|-------------|----------------|
| **Transisi tidak diizinkan** | `Status transition not allowed` | `Cannot change status from 'draft' to 'closed'. Allowed transitions: confirmed, cancelled` |
| **Tidak ada transisi untuk status saat ini** | `Status transition not allowed` | `No transitions defined for current status: 'unknown_status'` |

---

### 500 — Internal Server Error

**Definisi:** Error yang terjadi di sisi server dan **bukan disebabkan oleh kesalahan client**. Client tidak bisa memperbaiki error ini dengan mengubah request.

**Prinsip:** Ini adalah **catch-all terakhir** di setiap endpoint. Semua error yang tidak cocok dengan pola 400/404/409 masuk ke sini.

**Scope:**

| Kategori | Contoh Penyebab |
|----------|-----------------|
| **Database error** | Connection timeout, query syntax error, database down |
| **Lock failure** | `Failed to acquire lock for update operation. The resource is currently busy, please try again later.` |
| **Verification failure** | `Could not verify data existence before delete` |
| **Error tak terduga** | Runtime error, out of memory, null pointer |

**Field `details`:** Hanya muncul jika `NODE_ENV=development`. Berisi pesan error asli untuk debugging.

---

### 502 — Bad Gateway

**Definisi:** Hook API call eksternal dengan `blocking: true` mengembalikan error, sehingga transaksi di-rollback.

**Scope:** Hanya endpoint `/change-status` dengan workflow hook blocking.

| Kategori | Error Field | Contoh Message |
|----------|-------------|----------------|
| **Hook gagal** | `Workflow hook failed` | `Workflow onAfter hook failed: Blocking hook failed: HTTP 500: Internal Server Error` |

---

### Mapping PostgreSQL Error Code

| PG Error Code | Endpoint Create/Update | Endpoint Delete | Kategori |
|:-------------:|:---------------------:|:---------------:|----------|
| `23505` | 409 | 409 | Unique constraint violation |
| `23503` | **400** | **409** | Foreign key constraint. Perbedaan kode karena konteks berbeda |

**Alasan perbedaan `23503`:**
- **Create/Update:** data referensi tidak ada → kesalahan input dari client → **400**
- **Delete:** data masih direferensikan → konflik state database → **409**

---

## Format WHERE Clause

WHERE clause digunakan oleh beberapa endpoint (`/read`, `/delete`, `/datatables`, `/lookup`) untuk memfilter data. Tersedia dua format:

### Format Sederhana (Simple Array)

Seluruh kondisi digabungkan dengan `AND`, operator default `=`:

```json
{
  "where": [
    { "key": "is_active", "value": "true" },
    { "key": "company_id", "value": "uuid-value" }
  ]
}
```

Dihasilkan SQL:
```sql
WHERE is_active = $1 AND company_id = $2
```

### Format Kompleks (Complex Object)

Mendukung operator lengkap dan logika `AND`/`OR`:

```json
{
  "where": {
    "logic": "AND",
    "conditions": [
      { "key": "selling_price", "operator": ">=", "value": 10000 },
      { "key": "product_name", "operator": "LIKE", "value": "baut", "sensitive": false },
      { "key": "created_at", "operator": "BETWEEN", "value": ["2026-01-01", "2026-12-31"] },
      { "key": "uom", "operator": "IN", "value": ["PCS", "BOX"] },
      { "key": "sku", "operator": "IS NOT NULL" }
    ]
  }
}
```

### Operator yang Didukung (Supported Operators)

| Operator | Keterangan | Contoh Value |
|----------|------------|-------------|
| `=` | Sama dengan (default) | `"active"` |
| `!=` atau `<>` | Tidak sama dengan | `"deleted"` |
| `>` | Lebih besar dari | `100` |
| `<` | Lebih kecil dari | `50` |
| `>=` | Lebih besar atau sama | `10000` |
| `<=` | Lebih kecil atau sama | `99999` |
| `LIKE` | Pattern matching | `"baut"` (otomatis ditambah `%`) |
| `NOT LIKE` | Negasi pattern matching | `"draft"` |
| `BETWEEN` | Antara dua nilai | `["2026-01-01", "2026-12-31"]` |
| `NOT BETWEEN` | Di luar dua nilai | `[0, 100]` |
| `IN` | Salah satu dari daftar | `["PCS", "BOX", "KG"]` |
| `NOT IN` | Bukan salah satu dari daftar | `["deleted", "archived"]` |
| `IS NULL` | Bernilai null | (tidak perlu `value`) |
| `IS NOT NULL` | Tidak null | (tidak perlu `value`) |

**Opsi tambahan:**
- `"sensitive": false` untuk pencarian case-insensitive (membungkus kolom dan value dengan `UPPER()`)

---

## Format Sort Columns

Parameter `sort_columns` tersedia di endpoint `/read`, `/datatables`, `/lookup`, `/export`, dan `/aggregate`:

```json
{
  "sort_columns": [
    { "column": "supplier_name", "direction": "ASC" },
    { "column": "created_at", "direction": "DESC" }
  ]
}
```

| Property | Tipe | Wajib | Default | Keterangan |
|----------|------|:-----:|---------|------------|
| `column` | string | Ya | — | Nama kolom (harus ada di `validFields` payload) |
| `direction` | string | Tidak | `"ASC"` | Arah pengurutan: `"ASC"` atau `"DESC"` |

**Aturan:**
- Item yang kolomnya tidak ada di `validFields`, kosong, atau direction-nya bukan `ASC`/`DESC` akan **diabaikan** (silent skip) per item
- Namun jika **seluruh** item tidak valid (tidak tersisa satu pun kolom valid), endpoint mengembalikan **HTTP 400** dengan pesan `No valid sort columns provided`. Silent skip hanya berlaku selama masih ada minimal satu kolom valid
- Jika `sort_columns` kosong atau tidak dikirim, pengurutan menggunakan primary key ASC
- Item pertama dalam array = prioritas pengurutan tertinggi
- Direction bersifat case-insensitive (`"asc"` dan `"ASC"` sama)

---

## Referensi Silang (Cross Reference)

| Dokumen | Lokasi | Keterangan |
|---------|--------|------------|
| RDF Catalog | [../catalogs/rdf/](../catalogs/rdf/) | Spec Resource Definition File: field validation, query declarative, audit columns |
| Field Validation Catalog | [../commands/restforge-backend/catalog/field-validation.md](../commands/restforge-backend/catalog/field-validation.md) | Aturan validasi deklaratif dan pesan error |
| Commands Reference | [../commands/](../commands/) | Reference perintah CLI yang berlaku |
| Distributed Lock | [../features/distributed-lock/](../features/distributed-lock/) | Per-record locking dan resource lock |
