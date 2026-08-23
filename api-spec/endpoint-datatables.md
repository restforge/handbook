# DataTables — Server-Side Processing

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/datatables` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `200 OK` |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Format Response** | `{ draw, recordsTotal, recordsFiltered, data }` tanpa envelope `success` |
| **Paginasi** | `start` (offset) + `length` (limit, maks 1000) |
| **Pencarian** | `search.value` untuk teks, `searchBy` untuk kolom spesifik |
| **Sorting** | `sort_columns` array, mendukung multi-kolom |
| **Cache** | Redis (key: `rf:{project}:{endpoint}:datatables:{hash}`) |
| **Default Scope** | **Tidak** diterapkan |

---

## Ikhtisar

Endpoint `/datatables` dirancang untuk mengambil data dengan paginasi, pencarian, pengurutan, dan filter. Response menggunakan format yang kompatibel dengan library jQuery DataTables maupun custom frontend.

Berbeda dengan endpoint `/read`, datatables **tidak** menerapkan default scope sehingga administrator dapat melihat seluruh data termasuk yang tidak aktif.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/supplier/datatables
POST http://localhost:3000/api/mini-inventory/item-product/datatables
```

---

## Format Request

### Parameter

| Parameter | Tipe | Wajib | Default | Batasan | Keterangan |
|-----------|------|:-----:|---------|---------|------------|
| `draw` | number | Tidak | `1` | — | Nomor sequence request, dikembalikan di response |
| `start` | number | Tidak | `0` | >= 0 | Offset data (index awal) |
| `length` | number | Tidak | `10` | 1 — 1000 (auto-clamp) | Jumlah data per halaman |
| `search.value` | string | Tidak | `""` | — | Kata kunci pencarian teks |
| `searchBy` | string | Tidak | `"all"` | Harus ada di `datatablesFields` | Kolom target pencarian. `"all"` = semua kolom |
| `sort_columns` | array | Tidak | Primary key ASC | Kolom harus ada di `datatablesFields`; kolom tidak valid diabaikan. Lihat [Format Sort Columns](README.md#format-sort-columns) | Pengurutan data |
| `where` | array / object | Tidak | `null` | Key harus ada di `datatablesFields`; kolom tidak dikenal ditolak 400 (fail-closed). Lihat [Format WHERE](README.md#format-where-clause) | Kondisi filter kompleks |

**Format pencarian alternatif** (selain `search.value`):
- `searchValue` (alias untuk `search.value`)
- `search_value` (alias untuk `search.value`)

**Format sorting fallback** (kompatibilitas DataTables.net):
- `order[0][column]` (index kolom, number)
- `order[0][dir]` (arah pengurutan: `"asc"` / `"desc"`)

Prioritas resolusi sorting: `sort_columns` > `order[0][column]` > default (primary key ASC).

### Contoh Request

**1. Request dasar:**
```json
{
  "draw": 1,
  "start": 0,
  "length": 10,
  "search": { "value": "" }
}
```

**2. Halaman 2 dengan pencarian teks:**
```json
{
  "draw": 2,
  "start": 10,
  "length": 10,
  "search": { "value": "elektronik" }
}
```

**3. Pencarian di kolom spesifik dengan sorting:**
```json
{
  "draw": 1,
  "start": 0,
  "length": 25,
  "search": { "value": "PRD-001" },
  "searchBy": "product_code",
  "sort_columns": [
    { "column": "product_name", "direction": "ASC" }
  ]
}
```

**4. Dengan filter WHERE kompleks:**
```json
{
  "draw": 1,
  "start": 0,
  "length": 50,
  "search": { "value": "" },
  "where": {
    "logic": "AND",
    "conditions": [
      { "key": "is_active", "operator": "=", "value": "true" },
      { "key": "selling_price", "operator": ">=", "value": 10000 }
    ]
  }
}
```

---

## Format Response

### Response Sukses

**HTTP Status:** `200 OK`

```json
{
  "draw": 1,
  "recordsTotal": 150,
  "recordsFiltered": 42,
  "data": [
    {
      "rownumerator": 1,
      "supplier_id": "...",
      "supplier_code": "SUP-001",
      "supplier_name": "PT Maju Jaya",
      "is_active": true
    },
    {
      "rownumerator": 2,
      "supplier_id": "...",
      "supplier_code": "SUP-002",
      "supplier_name": "CV Berkah",
      "is_active": false
    }
  ]
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `draw` | number | Nomor sequence request (di-echo dari parameter input) |
| `recordsTotal` | number | Jumlah total **seluruh** data tanpa filter apa pun |
| `recordsFiltered` | number | Jumlah data setelah filter pencarian dan WHERE diterapkan |
| `data` | array | Array berisi data hasil query untuk halaman saat ini |
| `data[].rownumerator` | number | Nomor urut baris (dimulai dari `start + 1`) |

> **Penting:** Response datatables **tidak** menggunakan envelope `{ success: true }` karena format ini mengikuti standar jQuery DataTables.

### Response Error

#### 400 — Parameter Tidak Valid

Terjadi ketika query gagal dibangun karena parameter tidak valid (mis. format WHERE tidak valid). Response menggunakan `error: "Bad Request"` dengan `message` yang menjelaskan kesalahan:

```json
{
  "success": false,
  "error": "Bad Request",
  "message": "Invalid where conditions: ...",
  "details": "detail error teknis (hanya di development mode)",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 500 — Internal Server Error

Pesan 500 menyertakan nama endpoint hasil interpolasi (mis. `supplier`), bukan kata literal `datatables`:

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An error occurred while fetching supplier data",
  "details": "detail error teknis (hanya di development mode)",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

---

## Proyeksi Whitelist

Kolom yang dapat direferensikan di `sort_columns`, `searchBy`, dan `where` dibatasi oleh **`datatablesFields`**: proyeksi whitelist yang diturunkan saat generate dari irisan `fieldName` dengan kolom output `datatablesQuery`. Bila `datatablesQuery` tidak dikonfigurasi, proyeksi fallback ke `readableFields`. `datatablesFields` bersifat independen dari `readableFields` (`/read`/`/first`).

## Sumber Data

Endpoint `/datatables` menentukan sumber query dengan resolusi **2 tingkat**:

| Prioritas | Sumber | Keterangan |
|:---------:|--------|------------|
| 1 | `datatablesQuery` | Jika dikonfigurasi di payload, digunakan langsung sebagai base query |
| 2 | `readSource` | Fallback → `SELECT * FROM readSource`. Nilainya: `viewName` (jika ada) atau `tableName` |

> **Perbedaan dengan `/read`:** Endpoint `/datatables` **tidak** menggunakan `viewQuery` sebagai fallback. Jika `datatablesQuery` tidak dikonfigurasi, langsung fallback ke `readSource` (`viewName` atau `tableName`).

---

## Perbedaan dengan Endpoint /read

| Aspek | `/datatables` | `/read` |
|-------|---------------|---------|
| Format response | `{ draw, recordsTotal, recordsFiltered, data }` | `{ success, data, count, pagination }` |
| Field `success` | Tidak ada | Ada |
| Paginasi | `start` + `length` (offset-based) | `page` + `per_page` (page-based) |
| Default scope | **Tidak** diterapkan | Diterapkan (jika dikonfigurasi) |
| Sumber data | `datatablesQuery` → `readSource` (2 level) | `viewName` → `viewQuery` → `tableName` (3 level) |
| Whitelist kolom | `datatablesFields` | `readableFields` |
| Row numerator | Otomatis ditambahkan (`rownumerator`) | Tidak ada |
| Mode non-paginasi | Tidak ada (selalu paginasi) | Ada (jika `page` tidak dikirim) |

---

## Perilaku Cache

| Aspek | Nilai |
|-------|-------|
| Di-cache | Ya (jika Redis aktif dan `CACHE_ENABLED=true`) |
| Cache key | `rf:{project}:{endpoint}:datatables:{hash}` |
| Hash | MD5 (8 karakter) dari options request |
| TTL | Default 300 detik (configurable via `CACHE_TTL`) |
| Invalidasi | Otomatis setelah operasi `create`, `update`, atau `delete` |

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| DataTables (Deep Dive) | — | Dokumentasi lengkap termasuk WHERE operators dan contoh lanjutan |
| Sort Column | — | Format dan perilaku pengurutan data |
| Cache | — | Konfigurasi dan perilaku cache Redis |
| Query Declarative | — | Konfigurasi datatablesQuery dan sumber data |
