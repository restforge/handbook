# Read — Endpoint Pengambilan Data

---

## Referensi Cepat (Quick Reference)

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/read` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `200 OK` |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Mode Operasi** | Paginasi (dengan `page`) dan Non-Paginasi (tanpa `page`) |
| **Parameter Utama** | `page`, `per_page`, `limit`, `select`, `search_value`, `search_by`, `sort_columns`, `where` |
| **Cache** | Redis (key: `rf:{project}:{endpoint}:list:{hash}`) |
| **Default Scope** | Diterapkan jika dikonfigurasi di payload |

---

## Ikhtisar (Overview)

Endpoint `/read` adalah endpoint universal untuk mengambil data dengan dukungan **dua mode operasi**: paginasi dan non-paginasi. Mode ditentukan secara otomatis berdasarkan keberadaan parameter `page` di request body. Jika `page` dikirim maka mode paginasi aktif, jika tidak maka mode non-paginasi yang digunakan.

Endpoint ini mendukung pencarian teks, pengurutan multi-kolom, kondisi WHERE kompleks, dan pemilihan kolom selektif (`select`). Sumber data ditentukan berdasarkan prioritas resolusi: `viewName` → `viewQuery` → `tableName`.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/supplier/read
POST http://localhost:3000/api/mini-inventory/item-product/read
```

---

## Mode Operasi (Operation Modes)

| Mode | Kondisi | Keterangan |
|------|---------|------------|
| **Paginasi** | `page` dikirim di request body | Data per halaman, response memiliki blok `pagination` |
| **Non-Paginasi** | `page` **tidak** dikirim | Semua data yang cocok dikembalikan (dengan safety limit), tanpa blok `pagination` |

Tidak perlu parameter tambahan untuk memilih mode. Cukup kirim atau tidak kirim `page`.

---

## Format Request (Request Format)

### Parameter (Parameters)

| Parameter | Tipe | Wajib | Default | Batasan | Keterangan |
|-----------|------|:-----:|---------|---------|------------|
| `page` | number | Tidak | — | >= 1 | Nomor halaman. Jika dikirim → mode paginasi |
| `per_page` | number | Tidak | `10` | 1 — 100 | Jumlah data per halaman. Hanya berlaku jika `page` dikirim |
| `limit` | number | Tidak | `1000` | 1 — 5000 (auto-clamp) | Batas record di mode non-paginasi. Diabaikan jika `page` dikirim |
| `select` | array | Tidak | Semua kolom | Elemen harus ada di `validFields` | Kolom spesifik yang ditampilkan |
| `search_value` | string | Tidak | `""` | Maks 255 karakter | Kata kunci pencarian teks |
| `search_by` | string | Tidak | Kolom pertama payload | Harus ada di `validFields` | Kolom target pencarian |
| `sort_columns` | array | Tidak | Primary key ASC | Lihat [Format Sort Columns](README.md#format-sort-columns) | Pengurutan data |
| `where` | array / object | Tidak | `null` | Lihat [Format WHERE](README.md#format-where-clause) | Kondisi filter data |

### Validasi Parameter (Parameter Validation)

| Parameter | Aturan | Perilaku jika Tidak Valid |
|-----------|--------|--------------------------|
| `page` | Harus >= 1 | Error 400: `"Page must be greater than 0"` |
| `per_page` | Harus 1 — 100 | Error 400: `"Per page must be between 1 and 100"` |
| `limit` | Auto-clamp ke 1 — 5000 | Tidak error, value otomatis disesuaikan |
| `select` | Setiap elemen dicek terhadap `validFields` | Error 400: `"Invalid select fields"` dengan daftar `validFields` |
| `search_value` | Maks 255 karakter | Error 400: `"Search value must not exceed 255 characters"` |
| `search_by` | Harus ada di `validFields` | Error 400: `"Invalid search field"` |
| `sort_columns` | Kolom dicek terhadap `validFields` | Kolom tidak valid diabaikan (silent skip) |

### Contoh Request (Request Examples)

**1. Paginasi sederhana:**
```json
{
  "page": 1,
  "per_page": 20
}
```

**2. Paginasi dengan pencarian dan pengurutan:**
```json
{
  "page": 1,
  "per_page": 10,
  "search_value": "maju",
  "search_by": "supplier_name",
  "sort_columns": [
    { "column": "supplier_name", "direction": "ASC" }
  ]
}
```

**3. Non-paginasi dengan filter WHERE:**
```json
{
  "limit": 500,
  "where": [
    { "key": "is_active", "value": "true" }
  ],
  "sort_columns": [
    { "column": "created_date", "direction": "DESC" }
  ]
}
```

**4. Dengan kolom selektif:**
```json
{
  "page": 1,
  "per_page": 50,
  "select": ["supplier_id", "supplier_code", "supplier_name"]
}
```

**5. Dengan WHERE kompleks:**
```json
{
  "page": 1,
  "per_page": 25,
  "where": {
    "logic": "AND",
    "conditions": [
      { "key": "selling_price", "operator": ">=", "value": 10000 },
      { "key": "category_id", "operator": "IN", "value": ["cat-1", "cat-2"] },
      { "key": "product_name", "operator": "LIKE", "value": "baut", "sensitive": false }
    ]
  }
}
```

---

## Format Response (Response Format)

### Response Sukses — Mode Paginasi (Success Response — Pagination Mode)

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "data": [
    { "supplier_id": "...", "supplier_code": "SUP-001", "supplier_name": "PT Maju Jaya", "is_active": true },
    { "supplier_id": "...", "supplier_code": "SUP-002", "supplier_name": "CV Berkah", "is_active": true }
  ],
  "count": 2,
  "pagination": {
    "current_page": 1,
    "per_page": 10,
    "total_records": 42,
    "total_pages": 5,
    "has_next": true,
    "has_previous": false
  }
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `success` | boolean | `true` jika query berhasil |
| `data` | array | Array berisi data hasil query |
| `count` | number | Jumlah record di halaman ini (`data.length`) |
| `pagination.current_page` | number | Halaman yang sedang ditampilkan |
| `pagination.per_page` | number | Jumlah data per halaman |
| `pagination.total_records` | number | Total record setelah filter diterapkan |
| `pagination.total_pages` | number | Total halaman tersedia |
| `pagination.has_next` | boolean | `true` jika ada halaman berikutnya |
| `pagination.has_previous` | boolean | `true` jika ada halaman sebelumnya |

### Response Sukses — Mode Non-Paginasi (Success Response — Non-Pagination Mode)

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "data": [
    { "supplier_id": "...", "supplier_code": "SUP-001", "supplier_name": "PT Maju Jaya" },
    { "supplier_id": "...", "supplier_code": "SUP-002", "supplier_name": "CV Berkah" }
  ],
  "count": 2
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `success` | boolean | `true` jika query berhasil |
| `data` | array | Array berisi data hasil query |
| `count` | number | Jumlah record yang dikembalikan |

> **Catatan:** Blok `pagination` **tidak muncul** di mode non-paginasi.

### Response Error (Error Response)

#### 400 — Parameter Tidak Valid

Validasi parameter paginasi dan pencarian (`page`, `per_page`, `search_value`, `search_by`) menghasilkan format khusus **tanpa** field `error` dan **tanpa** `timestamp`. Key di dalam object `errors` adalah nama parameter terkait:

```json
{
  "success": false,
  "message": "Validation failed",
  "errors": {
    "page": ["Page must be greater than 0"]
  }
}
```

Contoh isi `errors` lainnya (key menyesuaikan parameter yang gagal):
- `"per_page": ["Per page must be between 1 and 100"]`
- `"search_value": ["Search value must not exceed 255 characters"]`
- `"search_by": ["Invalid search field. Valid fields: supplier_id, supplier_code, ..."]`

#### 400 — WHERE / Sort Tidak Valid

Error yang muncul saat membangun query (mis. format WHERE atau seluruh kolom sort tidak valid) dikembalikan dengan format berikut:

```json
{
  "success": false,
  "error": "Bad Request",
  "message": "Invalid where conditions: ...",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Field Select Tidak Valid

```json
{
  "success": false,
  "error": "Invalid select fields",
  "message": "Invalid field(s): field_tidak_ada",
  "validFields": ["supplier_id", "supplier_code", "supplier_name", "is_active"],
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 500 — Internal Server Error

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An error occurred while fetching supplier list data",
  "details": "detail error teknis (hanya di development mode)",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

---

## Sumber Data (Data Source)

Endpoint `/read` menentukan sumber query berdasarkan prioritas resolusi tiga tingkat:

| Prioritas | Properti Payload | Keterangan |
|:---------:|------------------|------------|
| 1 | `viewName` | Jika didefinisikan dan berbeda dari `tableName` → `SELECT * FROM viewName` |
| 2 | `viewQuery` | Digunakan jika `viewName` tidak ada dan berfungsi sebagai "virtual view" |
| 3 | `tableName` | Fallback terakhir → `SELECT * FROM tableName` |

> **Catatan:** `datatablesQuery` **tidak pernah** digunakan oleh endpoint `/read`. Query tersebut hanya untuk `/datatables`.

---

## Perilaku Cache (Cache Behavior)

| Aspek | Nilai |
|-------|-------|
| Di-cache | Ya (jika Redis aktif dan `CACHE_ENABLED=true`) |
| Cache key | `rf:{project}:{endpoint}:list:{hash}` |
| Hash | MD5 (8 karakter) dari options request |
| TTL | Default 300 detik (configurable via `CACHE_TTL`) |
| Invalidasi | Otomatis setelah operasi `create`, `update`, atau `delete` |

---

## Default Scope

Jika payload mengonfigurasi `defaultScope.read`, kondisi WHERE otomatis ditambahkan ke setiap query `/read`.

**Contoh konfigurasi payload:**
```json
{
  "defaultScope": {
    "read": { "is_active": true }
  }
}
```

**SQL yang dihasilkan:**
```sql
SELECT * FROM supplier WHERE is_active = true AND ... (user filters)
```

Default scope diterapkan **sebelum** kondisi WHERE dari request body, sehingga filter dari pengguna ditambahkan sebagai kondisi tambahan.

---

## Fitur Terkait (Related Features)

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| Read (Deep Dive) | — | Dokumentasi lengkap endpoint read termasuk resolusi sumber data |
| Sort Column | — | Format dan perilaku pengurutan data |
| Cache | — | Konfigurasi dan perilaku cache Redis |
| Default Scope | — | Filter otomatis yang diterapkan pada setiap query |
| Query Declarative | — | Konfigurasi viewName, viewQuery, dan sumber data |
