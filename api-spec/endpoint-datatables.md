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
| **Format Response** | `{ draw, recordsTotal, recordsFiltered, data, timestamp }` tanpa envelope `success` |
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
| `searchBy` | string | Tidak | `"all"` | Harus terdaftar di `datatablesWhere` payload; nilai tidak dikenal ditolak 400 | Kolom target pencarian. `"all"` = semua kolom pencarian |
| `sort_columns` | array | Tidak | Primary key ASC | Kolom harus ada di `datatablesFields`; kolom tidak valid diabaikan. Lihat [Format Sort Columns](README.md#format-sort-columns) | Pengurutan data |
| `where` | array / object | Tidak | `null` | Key harus ada di `datatablesFields`; kolom tidak dikenal ditolak 400 (fail-closed). Mendukung grup bersarang, lihat [Format WHERE](README.md#format-where-clause) | Kondisi filter kompleks |

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
  ],
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `draw` | number | Nomor sequence request (di-echo dari parameter input) |
| `recordsTotal` | number | Jumlah total **seluruh** data tanpa filter apa pun |
| `recordsFiltered` | number | Jumlah data setelah filter pencarian dan WHERE diterapkan |
| `data` | array | Array berisi data hasil query untuk halaman saat ini |
| `data[].rownumerator` | number | Nomor urut baris (dimulai dari `start + 1`) |
| `timestamp` | string | Waktu server saat response dikirim (ISO 8601 UTC). Dipakai aplikasi hasil generate sebagai acuan "hari ini" untuk badge `New` di halaman list; selalu dihitung saat response dikirim, tidak berasal dari cache |

> **Penting:** Response datatables **tidak** menggunakan envelope `{ success: true }` karena format ini mengikuti standar jQuery DataTables.

Field `timestamp` tersedia mulai rilis platform yang memuat perubahan ini. Aplikasi yang berjalan di atas rilis sebelumnya menerima response tanpa field itu, dan satu-satunya akibatnya badge `New` pada halaman list tidak dirender; perilaku list yang lain tidak berubah.

### Response Error

#### 400 — Nilai Tanggal-Waktu Tidak Valid

Terjadi ketika nilai field `date`, `timestamp`, atau `timestamptz` pada filter `where` tidak cocok dengan pola aktif (`DATEFORMAT`/`DATETIMEFORMAT`) maupun bentuk ISO. Tanggal saja pada field `timestamp`/`timestamptz`, tanggal kalender yang tidak nyata, jam di luar rentang, dan jam lokal pada celah atau tumpang-tindih DST termasuk kasus ini.

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

#### 400 — searchBy Tidak Dikenal

Terjadi ketika `searchBy` dikirim bersama kata kunci pencarian, tetapi nilainya tidak terdaftar di `datatablesWhere` payload. Nilai kosong dan `"all"` tetap sah. Nilai tak dikenal ditolak, bukan dialihkan diam-diam ke `"all"` seperti pada rilis sebelumnya:

```json
{
  "success": false,
  "error": "Bad Request",
  "message": "Invalid searchBy: kategori. Valid searchBy values: all, product_code, sku, product_name",
  "details": "detail error teknis (hanya di development mode)",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

Daftar pada `message` berisi seluruh nilai `datatablesWhere` payload, ditambah `all` bila entri itu belum tercantum.

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

Kolom yang dapat direferensikan di `sort_columns` dan `where` dibatasi oleh **`datatablesFields`**: proyeksi whitelist yang diturunkan saat generate dari irisan `fieldName` dengan kolom output `datatablesQuery`. Bila `datatablesQuery` tidak dikonfigurasi, proyeksi fallback ke `readableFields`. `datatablesFields` bersifat independen dari `readableFields` (`/read`/`/first`).

Parameter `searchBy` memakai daftar yang berbeda. Nilainya divalidasi terhadap `datatablesWhere` di payload, yaitu daftar kolom pencarian yang sengaja dibuka untuk endpoint ini. Kolom yang ada di `datatablesFields` tetapi tidak terdaftar di `datatablesWhere` karena itu ditolak 400. Penerapan filter LIKE tetap dibatasi `datatablesFields`, sehingga entri `datatablesWhere` yang tidak ikut di-output query tidak menghasilkan kondisi pencarian. Aturan penulisan entri `datatablesWhere` ada di [`catalogs/rdf/data-source.md`](../catalogs/rdf/data-source.md).

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
