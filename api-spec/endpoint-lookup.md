# Lookup — Endpoint Data Dropdown dan Autocomplete

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `GET` (dynamic search) dan `POST` (static/filtered lookup) |
| **URL** | `/api/{project}/{endpoint}/lookup` |
| **Content-Type** | `application/json` (POST) |
| **HTTP Status Sukses** | `200 OK` |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Header Wajib** | `X-Request-Mode: dynamic` (GET) atau `X-Request-Mode: static` (POST) |
| **Format Data** | `{ id, text }`, atau `{ id, text, row }` saat request menyertakan `select` (`id`/`text` configurable via `fieldNameLookup`) |
| **Default Scope** | Diterapkan (hanya data aktif jika dikonfigurasi) |
| **Cache** | Redis, 3 tipe: `lookup_dynamic`, `lookup_static`, `lookup_filter` |

---

## Ikhtisar

Endpoint `/lookup` dirancang untuk menyediakan data ringan ke komponen UI seperti dropdown, Select2, combobox, atau autocomplete. Setiap record dikembalikan minimal sebagai pasangan `{ id, text }` yang dapat langsung digunakan oleh frontend. Bila request menyertakan `select`, tiap record mendapat tambahan properti `row` berisi kolom mentah yang diminta — lihat [Kolom `row`](#kolom-row).

Endpoint ini mendukung dua HTTP method dengan perilaku berbeda:

| Method | Header | Kegunaan |
|--------|--------|----------|
| **GET** | `X-Request-Mode: dynamic` | Pencarian real-time saat pengguna mengetik (autocomplete) |
| **POST** | `X-Request-Mode: static` | Memuat seluruh data dropdown atau data dengan filter spesifik |

**Contoh URL:**
```
GET  http://localhost:3000/api/mini-inventory/supplier/lookup?search=maju
POST http://localhost:3000/api/mini-inventory/supplier/lookup
```

---

## GET — Pencarian Dinamis

### Header Wajib

| Header | Nilai | Keterangan |
|--------|-------|------------|
| `X-Request-Mode` | `dynamic` | **Wajib**. Request ditolak jika header tidak ada atau bernilai lain |

### Parameter

Parameter dikirim via query string:

| Parameter | Tipe | Wajib | Default | Batasan | Keterangan |
|-----------|------|:-----:|---------|---------|------------|
| `search` | string | Tidak | `""` | Maks 100 karakter | Kata kunci pencarian (case-insensitive) |
| `select` | string (JSON array) | Tidak | — | Harus berupa string hasil `JSON.stringify` dari array, mis. `["supplier_id","supplier_name","credit_limit"]` di-encode jadi `%5B%22supplier_id%22%2C%22supplier_name%22%2C%22credit_limit%22%5D` | Kolom tambahan yang ikut diminta ke lookup. Bila diisi, tiap item response mendapat `row` — lihat [Kolom `row`](#kolom-row) |
| `company_id` | string | Tidak | — | — | Filter tambahan (contoh; parameter extra bisa bervariasi per endpoint) |

### Contoh Request

**1. Pencarian dinamis:**
```
GET /api/mini-inventory/supplier/lookup?search=maju
X-Request-Mode: dynamic
```

**2. Pencarian dengan filter tambahan:**
```
GET /api/mini-inventory/supplier/lookup?search=jaya&company_id=uuid-company
X-Request-Mode: dynamic
```

**3. Tanpa search (load awal):**
```
GET /api/mini-inventory/supplier/lookup
X-Request-Mode: dynamic
```

**4. Dengan kolom tambahan (`select`):**
```
GET /api/mini-inventory/supplier/lookup?search=maju&select=%5B%22supplier_id%22%2C%22supplier_name%22%2C%22credit_limit%22%5D
X-Request-Mode: dynamic
```

### Response Sukses

**HTTP Status:** `200 OK`

Tanpa `select` (bentuk lama, tidak berubah):

```json
{
  "success": true,
  "count": 3,
  "data": [
    { "id": "uuid-1", "text": "PT Maju Jaya" },
    { "id": "uuid-2", "text": "PT Maju Sentosa" },
    { "id": "uuid-3", "text": "CV Maju Bersama" }
  ],
  "search": "maju",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

Dengan `select` (contoh request 4 di atas), tiap item mendapat tambahan `row`:

```json
{
  "success": true,
  "count": 1,
  "data": [
    {
      "id": "uuid-1",
      "text": "PT Maju Jaya",
      "row": { "supplier_id": "uuid-1", "supplier_name": "PT Maju Jaya", "credit_limit": "50000000.00" }
    }
  ],
  "search": "maju",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

### Response Error

#### 400 — Header X-Request-Mode Tidak Valid

```json
{
  "success": false,
  "error": "Invalid Request Mode",
  "message": "X-Request-Mode header must be set to dynamic",
  "details": "Contoh penggunaan: Tambahkan header X-Request-Mode: dynamic",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Parameter `select` Tidak Valid

Muncul bila `select` bukan string JSON yang valid, atau valid JSON tetapi
bukan array:

```json
{
  "success": false,
  "error": "Invalid select parameter",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Parameter Search Terlalu Panjang

```json
{
  "success": false,
  "error": "Invalid search parameter",
  "message": "Search parameter terlalu panjang (max 100 karakter)",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 500 — Internal Server Error

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An error occurred while looking up supplier data",
  "details": "detail error teknis (hanya di development mode)",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

---

## POST — Lookup Statis dan Lookup dengan Filter

### Header Wajib

| Header | Nilai | Keterangan |
|--------|-------|------------|
| `X-Request-Mode` | `static` | **Wajib**. Request ditolak jika header tidak ada atau bernilai lain |

### Mode POST

Endpoint POST memiliki tiga mode berdasarkan isi request body:

| Mode | Kondisi | Kegunaan |
|------|---------|----------|
| **Filtered lookup** | Body berisi `where` | Lookup dengan kondisi filter spesifik |
| **Static dengan select/sort** | Body berisi `select` atau `sort_columns` (tanpa `where`) | Muat semua data dengan kolom atau urutan custom |
| **Static legacy** | Body berisi `selected_tag` saja (atau body minimal) | Muat semua data, tandai item yang dipilih |

---

### Mode 1: Filtered Lookup (dengan `where`)

#### Parameter

| Parameter | Tipe | Wajib | Keterangan |
|-----------|------|:-----:|------------|
| `where` | array / object | **Ya** (untuk mode ini) | Kondisi filter. Key harus ada di `readableFields`; kolom tidak dikenal ditolak 400 (fail-closed). Lihat [Format WHERE](README.md#format-where-clause) |
| `select` | array | Tidak | Kolom yang ditampilkan. Field biasa divalidasi terhadap `readableFields`. Mendukung SQL expression (`\|\|`, `AS`, `CONCAT`) tanpa validasi |
| `sort_columns` | array | Tidak | Pengurutan data. Lihat [Format Sort Columns](README.md#format-sort-columns) |

#### Contoh Request

**1. Filter berdasarkan kolom:**
```json
// POST /api/mini-inventory/supplier/lookup
// X-Request-Mode: static
{
  "where": [
    { "key": "company_id", "value": "uuid-company" }
  ]
}
```

**2. Dengan kolom custom (SQL expression):**
```json
{
  "where": [
    { "key": "is_active", "value": "true" }
  ],
  "select": ["id", "supplier_code||' - '||supplier_name as display_text"]
}
```

**3. Dengan pengurutan:**
```json
{
  "where": [
    { "key": "company_id", "value": "uuid-company" }
  ],
  "select": ["id", "supplier_name"],
  "sort_columns": [
    { "column": "supplier_name", "direction": "ASC" }
  ]
}
```

**4. Dengan WHERE kompleks:**
```json
{
  "where": {
    "logic": "OR",
    "conditions": [
      { "key": "supplier_name", "operator": "LIKE", "value": "%Portal%" },
      { "key": "supplier_name", "operator": "LIKE", "value": "%system%", "sensitive": false }
    ]
  },
  "select": ["id", "supplier_code", "supplier_name", "email"]
}
```

#### Response Sukses

`select` request 2 di atas ber-alias (`... as display_text`) — nilai alias
ikut masuk `row` dengan key nama aliasnya:

```json
{
  "success": true,
  "count": 5,
  "data": [
    { "id": "uuid-1", "text": "SUP-001 - PT Maju Jaya", "row": { "id": "uuid-1", "display_text": "SUP-001 - PT Maju Jaya" } },
    { "id": "uuid-2", "text": "SUP-002 - CV Berkah Sentosa", "row": { "id": "uuid-2", "display_text": "SUP-002 - CV Berkah Sentosa" } }
  ],
  "query": {
    "where": [{ "key": "company_id", "value": "uuid-company" }],
    "select": ["id", "supplier_code||' - '||supplier_name as display_text"]
  },
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

> **Catatan:** Pada mode ini (`where` disertakan), response tetap membawa
> `row` walau `select` tidak diisi — isinya minimal hanya primary key
> (`{ "row": { "id": "..." } }`). Ini beda dari Mode 3 (`selected_tag`) di
> bawah, yang bentuknya tetap `{ id, text }` murni tanpa `row` sama sekali
> selama `select` tidak diisi.

---

### Mode 2: Static dengan Select/Sort (tanpa `where`)

#### Contoh Request

```json
// POST /api/mini-inventory/supplier/lookup
// X-Request-Mode: static
{
  "select": ["id", "supplier_name"],
  "sort_columns": [
    { "column": "supplier_name", "direction": "ASC" }
  ]
}
```

#### Response Sukses

```json
{
  "success": true,
  "count": 150,
  "data": [
    { "id": "uuid-1", "text": "CV Abadi Makmur", "row": { "id": "uuid-1", "supplier_name": "CV Abadi Makmur" } },
    { "id": "uuid-2", "text": "CV Berkah Sentosa", "row": { "id": "uuid-2", "supplier_name": "CV Berkah Sentosa" } }
  ],
  "selectedTag": "",
  "query": {
    "select": ["id", "supplier_name"],
    "sort_columns": [{ "column": "supplier_name", "direction": "ASC" }],
    "where": []
  },
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

---

### Mode 3: Static Legacy (dengan `selected_tag`)

#### Parameter

| Parameter | Tipe | Wajib | Keterangan |
|-----------|------|:-----:|------------|
| `selected_tag` | string | Tidak | ID record yang sedang dipilih. Item yang cocok akan diberi flag `selected: "true"` di response |

#### Contoh Request

```json
// POST /api/mini-inventory/supplier/lookup
// X-Request-Mode: static
{
  "selected_tag": "uuid-supplier-1"
}
```

#### Response Sukses

```json
{
  "success": true,
  "count": 150,
  "data": [
    { "id": "uuid-supplier-1", "text": "PT Maju Jaya", "selected": "true" },
    { "id": "uuid-supplier-2", "text": "CV Berkah Sentosa" },
    { "id": "uuid-supplier-3", "text": "PT Harmoni Abadi" }
  ],
  "selectedTag": "uuid-supplier-1",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

> **Catatan:** Hanya item yang `id`-nya cocok dengan `selected_tag` yang mendapat property `selected: "true"`.

---

### Response Error POST

#### 400 — Header X-Request-Mode Tidak Valid

```json
{
  "success": false,
  "error": "Invalid Request Mode",
  "message": "X-Request-Mode header must be set to static for POST method",
  "details": "Contoh penggunaan: Tambahkan header X-Request-Mode: static",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Payload Kosong

Response ini juga menyertakan contoh format request yang benar:

```json
{
  "success": false,
  "error": "Invalid Payload",
  "message": "Payload cannot be empty",
  "example": {
    "selected_tag": "optional-id-if-needed",
    "where": [{ "key": "code", "value": "AP2506-01" }],
    "select": ["id", "supplier_name"]
  },
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Format WHERE Tidak Valid

```json
{
  "success": false,
  "error": "Invalid where format",
  "message": "Invalid where format",
  "example": {
    "where": [{ "key": "code", "value": "AP2506-01" }],
    "select": ["id", "supplier_name"]
  },
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Field Select Tidak Valid

```json
{
  "success": false,
  "error": "Invalid select fields",
  "message": "Invalid field(s): field_tidak_ada",
  "validFields": ["supplier_id", "supplier_code", "supplier_name", "email"],
  "sqlExpressionNote": "SQL expressions dengan operator || atau AS alias diperbolehkan",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

> **Catatan:** Field `validFields` dalam respons error hanya disertakan saat `NODE_ENV=development`. Di produksi, field ini tidak ditampilkan (anti-disclosure).

> Validasi select memiliki **pengecualian**. Field yang mengandung SQL expression (`||`, `CONCAT`, `COALESCE`, `CASE`, `WHEN`) atau alias (`AS`) **tidak divalidasi** terhadap `readableFields`.

#### 500 — Internal Server Error

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

## Struktur Data Response

Setiap item di array `data` menggunakan format berikut:

| Field | Tipe | Keterangan |
|-------|------|------------|
| `id` | string | Primary key record |
| `text` | string | Teks yang ditampilkan di dropdown |
| `row` | object | Hanya muncul bila request menyertakan `select` (atau `where`, lihat catatan Mode 1). Berisi seluruh kolom yang diminta, termasuk primary key — lihat [Kolom `row`](#kolom-row) |
| `selected` | string | `"true"` hanya pada item yang cocok dengan `selected_tag` (opsional) |

### Kolom `row`

Ketika request menyertakan `select`, tiap item di `data[]` mendapat tambahan
properti `row`: objek berisi seluruh kolom yang diminta di `select` — termasuk
primary key, meski primary key tidak ditulis secara eksplisit di `select` — dengan
key persis sesuai nama kolom (atau nama alias, untuk kolom berbentuk SQL
expression `... AS alias`) yang ditulis di `select`. Tanpa `select`, response
tetap bentuk lama `{ id, text }` tanpa `row`, sehingga konsumen yang sudah ada
tidak perlu penyesuaian apa pun.

Kolom yang diminta di `select` tetap tunduk pada whitelist kolom yang berlaku
untuk endpoint ini (lihat [400 — Field Select Tidak Valid](#400--field-select-tidak-valid))
— kolom yang tidak diizinkan tetap ditolak, tidak pernah ikut masuk `row`.

Kolom yang dipakai sebagai `text` mengikuti aturan berikut bila
`fieldNameLookup.text` tidak dikonfigurasi secara eksplisit (lihat
[Konfigurasi Kolom Lookup](#konfigurasi-kolom-lookup) di bawah):

| Situasi | Kolom yang dipakai sebagai `text` |
|---------|-----------------------------------|
| `select` memuat kolom bertipe teks | Kolom bertipe teks pertama, sesuai urutan penulisan di `select` |
| `select` tidak memuat kolom bertipe teks sama sekali | Kolom non-primary-key pertama di `select` |

### Konfigurasi Kolom Lookup

Kolom `id` dan `text` dapat dikonfigurasi melalui `fieldNameLookup` di payload JSON:

```json
{
  "fieldNameLookup": {
    "id": "supplier_id",
    "text": "supplier_name"
  }
}
```

Jika `fieldNameLookup` tidak dikonfigurasi, default menggunakan kolom `id` dan kolom teks pertama di `fieldName`. Key pada `row` tidak terpengaruh konfigurasi ini — `row` selalu mengikuti nama kolom (atau alias) apa adanya seperti yang ditulis di `select`, terlepas dari nama kolom `id`/`text` yang dikonfigurasi `fieldNameLookup`.

---

## Perilaku Cache

| Aspek | Nilai |
|-------|-------|
| GET (dynamic) | **Tidak di-cache** — variasi keystroke terlalu tinggi |
| POST (static) | **Di-cache** — key: `rf:{project}:{endpoint}:lookup_static:{hash}` |
| POST (filtered) | **Di-cache** — key: `rf:{project}:{endpoint}:lookup_filter:{hash}` |
| Invalidasi | Otomatis setelah operasi `create`, `update`, atau `delete` |

---

## Default Scope

Lookup **selalu** menerapkan default scope jika dikonfigurasi di payload. Hal ini memastikan dropdown hanya menampilkan data yang aktif/valid.

**Contoh:** Jika payload mengonfigurasi `defaultScope.lookup: { is_active: true }`, maka semua query lookup mendapat kondisi `WHERE is_active = true` secara otomatis.

| Mode | Default Scope Diterapkan |
|------|:------------------------:|
| GET (dynamic) | Ya |
| POST (static) | Ya |
| POST (filtered) | Ya |

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| Default Scope | — | Filter otomatis agar lookup hanya menampilkan data aktif |
| Cache | — | Konfigurasi dan perilaku cache Redis (3 tipe lookup) |
| Sort Column | — | Pengurutan data dalam lookup filtered/static |
