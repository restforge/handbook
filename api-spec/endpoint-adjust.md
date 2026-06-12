# Adjust — Endpoint Penyesuaian Field Numerik

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/adjust` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `200 OK` |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Parameter Wajib** | Primary key + `adjustments` array |
| **Operasi** | Atomic increment/decrement (`SET field = field + N`) |
| **Guard Clause** | SQL-level constraint `(field + N) >= min` |
| **Cache** | Invalidasi otomatis setelah adjust berhasil |
| **Distributed Lock** | Per-record WRITE lock (jika diaktifkan) |
| **Event Lifecycle** | `onBeforeAdjust` → UPDATE → `onAfterAdjust` |

---

## Ikhtisar

Endpoint `/adjust` digunakan untuk melakukan penyesuaian atomik pada field numerik, misalnya menambah atau mengurangi stok. Operasi bersifat **atomik di level SQL** (`SET stock = stock + (-3)`), bukan read-modify-write, sehingga aman dari race condition pada akses concurrent.

Guard clause dapat dikonfigurasi untuk mencegah nilai hasil menjadi negatif (misalnya stok tidak boleh di bawah 0). Guard clause ini disisipkan langsung di WHERE clause SQL, sehingga validasi dan update terjadi dalam satu operasi atomik.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/item-product/adjust
```

---

## Format Request

### Parameter

| Parameter | Tipe | Wajib | Keterangan |
|-----------|------|:-----:|------------|
| `{primaryKey}` | string | **Ya** | Nilai primary key record yang akan di-adjust |
| `adjustments` | array | **Ya** | Array operasi adjustment (minimum 1 item) |
| `adjustments[].field` | string | **Ya** | Nama field yang di-adjust. Harus terdaftar di `adjustConfig.fields` |
| `adjustments[].value` | number | **Ya** | Delta value: positif = tambah, negatif = kurang. Harus non-zero |
| `reason` | string | Kondisional | Alasan adjustment. Wajib jika `adjustConfig.reasonRequired = true` |
| `updated_by` | string | Tidak | Pelaku adjustment (audit). Resolusi nilai berurutan: field ini di request body → header `x-user-id` → fallback `"system"` |

### Contoh Request

**1. Kurangi stok (decrement):**
```json
{
  "item_product_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "adjustments": [
    { "field": "stock", "value": -3 }
  ],
  "reason": "Stock outbound #OUT-2026-001"
}
```

**2. Tambah stok (increment):**
```json
{
  "item_product_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "adjustments": [
    { "field": "stock", "value": 50 }
  ],
  "reason": "Stock inbound #INB-2026-005"
}
```

**3. Multi-field adjustment:**
```json
{
  "item_product_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "adjustments": [
    { "field": "stock", "value": -10 },
    { "field": "min_stock", "value": 5 }
  ],
  "reason": "Penyesuaian stok dan minimum stok"
}
```

---

## Body Options (Opsional)

Endpoint `/adjust` mendukung format request body alternatif dengan wrapper `{data, options}`. Format ini memungkinkan pengiriman opsi tambahan yang dapat dibaca oleh component engine handler dan processor.

### Format dengan Options

```json
{
  "data": {
    "item_product_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "adjustments": [
      { "field": "stock", "value": -3 }
    ],
    "reason": "Stock outbound #OUT-2026-001"
  },
  "options": {
    "notify_warehouse": true,
    "source_document": "OUT-2026-001"
  }
}
```

### Aturan

- Body harus memiliki **tepat 2 key**: `data` dan `options`
- `data` harus berupa object (berisi primary key + adjustments array)
- `options` harus berupa object (berisi flag/parameter tambahan)
- Jika format ini tidak terdeteksi, body diproses sebagai format standar (backward compatible)
- Nilai `options` tersedia di component handler sebagai template variable `{options}` dan di processor melalui `req.bodyOptions`

---

## Format Response

### Response Sukses

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "message": "item-product data successfully adjusted",
  "data": {
    "item_product_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "stock": 97
  },
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `success` | boolean | `true` jika adjustment berhasil |
| `message` | string | `"{endpoint} data successfully adjusted"` |
| `data` | object | Berisi primary key dan field yang di-adjust (hanya field terkait, bukan seluruh record) |
| `timestamp` | string | Waktu eksekusi (ISO 8601) |

### Response Error

#### 400 — Payload Kosong

```json
{
  "success": false,
  "error": "Invalid payload",
  "message": "Payload cannot be empty",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Primary Key Tidak Ada

```json
{
  "success": false,
  "error": "Missing required field",
  "message": "Primary key (item_product_id) is required for adjust",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Adjustments Kosong atau Bukan Array

```json
{
  "success": false,
  "error": "Invalid payload",
  "message": "adjustments array is required and must not be empty",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Field Tidak Dikonfigurasi

```json
{
  "success": false,
  "error": "Validation error",
  "message": "Field \"unknown_field\" is not configured for adjustment",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Field Bukan Numerik

```json
{
  "success": false,
  "error": "Validation error",
  "message": "Field \"supplier_name\" has type \"string\" — only numeric fields (type: \"number\") can be adjusted",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Value Bukan Angka atau Nol

```json
{
  "success": false,
  "error": "Validation error",
  "message": "Adjustment value for \"stock\" must be a non-zero number",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Reason Wajib Tapi Kosong

```json
{
  "success": false,
  "error": "Validation error",
  "message": "reason is required for adjust operation",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 404 — Data Tidak Ditemukan

```json
{
  "success": false,
  "error": "Data not found",
  "message": "item-product data not found",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 409 — Constraint Violation (Guard Clause Gagal)

Terjadi ketika hasil adjustment akan berada di bawah batas minimum yang dikonfigurasi.

```json
{
  "success": false,
  "error": "Constraint violation",
  "message": "Adjust failed: record not found or constraint violation (value would go below minimum)",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 409 — Duplicate Entry

Terjadi bila adjustment melanggar unique constraint (PostgreSQL error code `23505`).
Berbeda dengan guard clause di atas, kasus ini mengembalikan `error: "Duplicate entry"`
disertai map `details`.

```json
{
  "success": false,
  "error": "Duplicate entry",
  "message": "A record with this value already exists",
  "details": {
    "sku": ["A record with this value already exists"]
  },
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

Field `details` adalah map `{ "<kolom>": ["<pesan>"] }` yang menunjuk kolom pelanggar
unique constraint (untuk constraint multi-kolom, tiap kolom dipetakan ke
`"A record with this combination of values already exists"`). Field ini selalu muncul
bila kolom pelanggar dapat diekstrak; bila tidak, field ini tidak disertakan.

#### 500 — Internal Server Error

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An error occurred while adjusting item-product data",
  "details": "detail error teknis (hanya di development mode)",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

---

## Guard Clause

Guard clause mencegah nilai field melampaui batas yang dikonfigurasi. Validasi ini disisipkan langsung di SQL WHERE clause, sehingga **atomik dan concurrent-safe**.

**SQL yang dihasilkan** (contoh: `stock - 3` dengan `min: 0`, `allowNegativeResult: false`):
```sql
UPDATE item_product
SET stock = stock + $1,
    updated_at = CURRENT_TIMESTAMP,
    updated_by = $2
WHERE item_product_id = $3
  AND (stock + -3) >= 0
RETURNING *
```

> **Catatan:** Delta value (`-3`) dikirim sebagai parameter `$1` pada klausa `SET`, sedangkan pada guard clause nilai tersebut disisipkan langsung ke ekspresi `(stock + -3)`. Kolom audit yang ditetapkan adalah `updated_at` dan `updated_by` (bukan kolom legacy `modified_by`/`modified_date`).

Jika `(stock + (-3)) >= 0` bernilai `false`, query menghasilkan 0 rows affected → response 409.

---

## Konfigurasi Payload

Endpoint `/adjust` memerlukan konfigurasi `adjustConfig` di payload JSON:

```json
{
  "action": { "adjust": true },
  "adjustConfig": {
    "reasonRequired": true,
    "fields": {
      "stock": {
        "type": "number",
        "min": 0,
        "allowNegativeResult": false
      },
      "min_stock": {
        "type": "number",
        "min": 0,
        "allowNegativeResult": false
      }
    }
  }
}
```

| Property | Tipe | Default | Keterangan |
|----------|------|---------|------------|
| `reasonRequired` | boolean | `false` | Jika `true`, parameter `reason` wajib diisi |
| `fields.{name}.type` | string | `"number"` | Harus `"number"` |
| `fields.{name}.min` | number | `0` | Nilai minimum yang diperbolehkan setelah adjustment |
| `fields.{name}.allowNegativeResult` | boolean | `true` | Jika `false`, guard clause ditambahkan ke SQL |

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| Adjust (Deep Dive) | — | Dokumentasi lengkap termasuk validasi multi-layer |
| Cache | — | Invalidasi cache setelah adjustment berhasil |
| Distributed Lock | — | Per-record WRITE lock saat adjustment |
| Component Engine | — | Event hooks (onBeforeAdjust, onAfterAdjust) |
| Body Options | — | Opsi tambahan via format `{data, options}` |
