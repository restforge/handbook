# Aggregate — Endpoint Agregasi Data

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/aggregate` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `200 OK` |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Fungsi** | `count`, `sum`, `avg`, `min`, `max` |
| **GROUP BY** | Didukung (response menjadi array) |
| **HAVING** | Didukung (memerlukan GROUP BY) |
| **JOIN** | Didukung via konfigurasi `aggregateConfig.joins` di payload |

---

## Ikhtisar

Endpoint `/aggregate` digunakan untuk menjalankan operasi agregasi data, yaitu menghitung jumlah record, menjumlahkan nilai, menghitung rata-rata, dan sebagainya. Mendukung GROUP BY untuk pengelompokan, HAVING untuk filter hasil agregasi, dan JOIN ke tabel lain yang dikonfigurasi di payload.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/item-product/aggregate
```

---

## Format Request

### Parameter

| Parameter | Tipe | Wajib | Default | Keterangan |
|-----------|------|:-----:|---------|------------|
| `operations` | array | Tidak | `[{ function: "count", field: "*" }]` | Operasi agregasi |
| `operations[].function` | string | Ya | — | `count`, `sum`, `avg`, `min`, `max` |
| `operations[].field` | string | Ya | — | Nama field tabel utama atau `"*"` (`"*"` hanya untuk `count`). Field hasil join tidak boleh dipakai di sini |
| `operations[].alias` | string | Tidak | `{function}_{field}` | Nama kolom hasil (alfanumerik + underscore) |
| `joins` | array | Tidak | `[]` | Nama join yang didefinisikan di `aggregateConfig` |
| `group_by` | array | Tidak | `[]` | Kolom untuk pengelompokan. Boleh field tabel utama atau field hasil join yang didefinisikan di `aggregateConfig` |
| `where` | object | Tidak | — | Kondisi filter. Lihat [Format WHERE](README.md#format-where-clause) |
| `having` | array | Tidak | `[]` | Filter hasil agregasi (memerlukan `group_by`) |
| `having[].function` | string | Ya | — | `count`, `sum`, `avg`, `min`, `max` |
| `having[].field` | string | Ya | — | Nama field tabel utama atau `"*"`. Field hasil join tidak boleh dipakai di sini, sama seperti `operations` |
| `having[].operator` | string | Ya | — | `=`, `<>`, `!=`, `>`, `<`, `>=`, `<=` |
| `having[].value` | number/string | Ya | — | Nilai perbandingan |
| `sort_columns` | array | Tidak | — | Pengurutan hasil. Lihat [Format Sort Columns](README.md#format-sort-columns) |
| `limit` | number | Tidak | `1000` | Maks 10000 |

> **Catatan field:** Field hasil join hanya boleh digunakan pada `group_by` dan `select`. Untuk `operations` dan `having`, field wajib berupa field tabel utama atau `"*"`.

### Contoh Request

**1. Count sederhana (tanpa parameter):**
```json
{}
```

**2. Multiple operasi:**
```json
{
  "operations": [
    { "function": "count", "field": "*", "alias": "total_records" },
    { "function": "sum", "field": "stock", "alias": "total_stock" },
    { "function": "avg", "field": "selling_price", "alias": "avg_price" },
    { "function": "min", "field": "stock", "alias": "min_stock" },
    { "function": "max", "field": "stock", "alias": "max_stock" }
  ]
}
```

**3. Dengan GROUP BY:**
```json
{
  "operations": [
    { "function": "count", "field": "*", "alias": "total" },
    { "function": "sum", "field": "stock", "alias": "total_stock" }
  ],
  "group_by": ["category_id"]
}
```

**4. Dengan JOIN, GROUP BY, dan HAVING:**
```json
{
  "operations": [
    { "function": "count", "field": "*", "alias": "total" },
    { "function": "sum", "field": "stock", "alias": "total_stock" }
  ],
  "joins": ["warehouse"],
  "group_by": ["warehouse_id", "warehouse_name"],
  "having": [
    { "function": "count", "field": "*", "operator": ">", "value": 10 }
  ],
  "sort_columns": [
    { "column": "total_stock", "direction": "DESC" }
  ]
}
```

**5. Dengan WHERE filter:**
```json
{
  "operations": [
    { "function": "sum", "field": "stock", "alias": "active_stock" }
  ],
  "where": {
    "logic": "AND",
    "conditions": [
      { "key": "is_active", "operator": "=", "value": "true" }
    ]
  }
}
```

---

## Format Response

### Response Sukses — Tanpa GROUP BY (Single Object)

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "data": {
    "total_records": 150,
    "total_stock": 5000,
    "avg_price": 33333.33,
    "min_stock": 0,
    "max_stock": 500
  },
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

### Response Sukses — Dengan GROUP BY (Array)

```json
{
  "success": true,
  "data": [
    {
      "category_id": "uuid-cat-1",
      "category_name": "Electronics",
      "total": 50,
      "total_stock": 2000
    },
    {
      "category_id": "uuid-cat-2",
      "category_name": "Clothing",
      "total": 45,
      "total_stock": 1500
    }
  ],
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

> **Catatan:** Tanpa `group_by`, `data` berupa single object. Dengan `group_by`, `data` berupa array.

### Response Error

#### 400 — Validation Error

Seluruh kegagalan validasi aggregate dikembalikan dengan field `error` yang seragam (`"Validation error"`). Hanya `message` yang berbeda sesuai jenis kesalahan:

```json
{
  "success": false,
  "error": "Validation error",
  "message": "Invalid aggregate function \"median\". Allowed: count, sum, avg, min, max",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

Contoh `message` lain pada status 400:
- `"Field \"unknown_field\" is not a valid field"` — field tidak valid
- `"Field \"*\" is only allowed with COUNT function"` — wildcard `*` dengan fungsi selain COUNT
- `"Join \"unknown_table\" is not defined in aggregate config"` — join tidak terdefinisi di `aggregateConfig`
- `"HAVING clause requires GROUP BY"` — HAVING tanpa GROUP BY

#### 403 — Pro Feature Required

Sebagian kapabilitas aggregate merupakan fitur Pro. Jika dipanggil tanpa lisensi yang sesuai, response menyertakan field `upgrade`:

```json
{
  "success": false,
  "error": "Pro Feature Required",
  "message": "...",
  "upgrade": "https://restforge.dev/pricing",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 500 — Internal Server Error

Pesan 500 menyertakan nama endpoint hasil interpolasi (mis. `supplier`):

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An error occurred while aggregating supplier data",
  "details": "detail error teknis (hanya di development mode)",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

---

## Konfigurasi Join

Join dikonfigurasi di payload JSON melalui `aggregateConfig.joins`:

```json
{
  "aggregateConfig": {
    "joins": {
      "warehouse": {
        "tableName": "warehouse",
        "joinType": "LEFT",
        "sourceField": "warehouse_id",
        "targetField": "warehouse_id",
        "fields": ["warehouse_code", "warehouse_name"]
      }
    }
  }
}
```

| Property | Tipe | Keterangan |
|----------|------|------------|
| `tableName` | string | Nama tabel yang di-join |
| `joinType` | string | `INNER`, `LEFT`, `RIGHT` |
| `sourceField` | string | Kolom di tabel utama |
| `targetField` | string | Kolom di tabel join |
| `fields` | array | Kolom dari tabel join yang tersedia hanya untuk `group_by` dan `select`. Tidak berlaku untuk `operations` maupun `having` |

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| Aggregate (Deep Dive) | — | Dokumentasi lengkap termasuk contoh SQL dan konfigurasi |
| Sort Column | — | Pengurutan hasil agregasi |
