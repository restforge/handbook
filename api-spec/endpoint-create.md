# Create — Endpoint Pembuatan Data Baru

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/create` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `201 Created` |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Validasi** | `fieldValidation` dari payload JSON (opsional) |
| **Cache** | Invalidasi otomatis setelah create berhasil |
| **Event Lifecycle** | `onBeforeInsert` → INSERT → `onAfterInsert` |

---

## Ikhtisar

Endpoint `/create` digunakan untuk menyisipkan satu record baru ke dalam tabel database. Request body berisi pasangan field-value yang akan di-insert. Primary key akan di-generate secara otomatis sebagai UUIDv7 jika tidak disertakan, bernilai kosong, atau diisi `"-"`.

Endpoint ini mendukung validasi deklaratif melalui konfigurasi `fieldValidation` di payload JSON, sehingga setiap field dapat divalidasi tanpa menulis kode tambahan. Setelah operasi berhasil, seluruh cache terkait endpoint ini akan diinvalidasi secara otomatis.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/supplier/create
POST http://localhost:3000/api/mini-inventory/item-product/create
```

---

## Format Request

### Parameter

Request body berisi JSON object dengan pasangan field-value. Field yang diterima ditentukan oleh konfigurasi `fieldName` dalam payload JSON endpoint.

```json
{
  "supplier_code": "SUP-001",
  "supplier_name": "PT Maju Jaya",
  "email": "contact@majujaya.com",
  "phone": "021-5551234",
  "address": "Jl. Industri No. 10",
  "is_active": true
}
```

**Aturan:**
- Hanya field yang terdaftar di `fieldName` payload yang akan diproses. Field lain diabaikan
- Primary key boleh disertakan atau tidak (auto UUID jika kosong/"-"/tidak ada)
- String kosong `""` dikonversi secara otomatis ke `null` pada field yang namanya mengandung `date`, `time`, `timestamp`, `tags`, atau `json`
- Field datetime diurai sesuai konfigurasi `dateTimeFields` di payload

### Contoh Request

**1. Create tanpa primary key (auto UUID):**
```json
{
  "supplier_code": "SUP-001",
  "supplier_name": "PT Maju Jaya",
  "is_active": true
}
```

**2. Create dengan primary key eksplisit:**
```json
{
  "supplier_id": "550e8400-e29b-41d4-a716-446655440000",
  "supplier_code": "SUP-002",
  "supplier_name": "CV Berkah Sentosa"
}
```

**3. Create dengan field datetime:**
```json
{
  "contract_code": "CTR-2026-001",
  "start_date": "2026-04-01",
  "end_date": "2027-03-31",
  "notes": "Kontrak tahunan"
}
```

---

## Body Options (Opsional)

Endpoint `/create` mendukung format request body alternatif dengan wrapper `{data, options}`. Format ini memungkinkan pengiriman opsi tambahan yang dapat dibaca oleh component engine handler dan processor.

### Format dengan Options

```json
{
  "data": {
    "supplier_code": "SUP-001",
    "supplier_name": "PT Maju Jaya",
    "email": "contact@majujaya.com",
    "is_active": true
  },
  "options": {
    "send_feedback_email": true,
    "send_to_kafka": false
  }
}
```

### Aturan

- Body harus memiliki **tepat 2 key**: `data` dan `options`
- `data` harus berupa object (berisi field-value yang akan di-insert)
- `options` harus berupa object (berisi flag/parameter tambahan)
- Jika format ini tidak terdeteksi, body diproses sebagai format standar (backward compatible)
- Nilai `options` tersedia di component handler sebagai template variable `{options}` dan di processor melalui `req.bodyOptions`

---

## Format Response

### Response Sukses

**HTTP Status:** `201 Created`

```json
{
  "success": true,
  "message": "supplier data successfully added",
  "data": {
    "supplier_id": "550e8400-e29b-41d4-a716-446655440000",
    "supplier_code": "SUP-001",
    "supplier_name": "PT Maju Jaya",
    "email": "contact@majujaya.com",
    "phone": "021-5551234",
    "address": "Jl. Industri No. 10",
    "is_active": true,
    "created_at": "2026-03-30T10:30:00.000Z",
    "created_by": "system",
    "updated_at": "2026-03-30T10:30:00.000Z",
    "updated_by": "system"
  },
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `success` | boolean | `true` jika insert berhasil |
| `message` | string | `"{endpoint} data successfully added"` |
| `data` | object | Record yang baru dibuat (termasuk field auto-populated) |
| `timestamp` | string | Waktu eksekusi (ISO 8601) |

### Response Error

#### 400 — Payload Kosong

Terjadi ketika body request kosong atau tidak berisi data yang valid.

```json
{
  "success": false,
  "error": "Invalid payload",
  "message": "Payload cannot be empty",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Validasi Gagal

Terjadi ketika data tidak memenuhi constraint yang didefinisikan di `fieldValidation`.

```json
{
  "success": false,
  "error": "Validation failed",
  "message": "Invalid data",
  "errors": {
    "supplier_code": ["Field supplier_code is required"],
    "email": ["Field email must be a valid email address"],
    "supplier_name": ["Field supplier_name must be at most 100 characters"]
  },
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Foreign Key Constraint

Terjadi ketika data mereferensikan record yang tidak ada di tabel relasi (PostgreSQL error code `23503`).

```json
{
  "success": false,
  "error": "Foreign key constraint",
  "message": "Referenced data not found",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 409 — Duplicate Entry

Terjadi ketika data melanggar unique constraint (PostgreSQL error code `23505`).

```json
{
  "success": false,
  "error": "Duplicate entry",
  "message": "A record with this value already exists",
  "details": {
    "email": ["A record with this value already exists"]
  },
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

Field `details` adalah map `{ "<kolom>": ["<pesan>"] }` yang menunjuk kolom pelanggar
unique constraint, berguna untuk memetakan error ke field form. Untuk unique constraint
multi-kolom, setiap kolom dipetakan ke pesan
`"A record with this combination of values already exists"`. Berbeda dengan `details`
pada 500 (string, hanya `NODE_ENV=development`), `details` pada 409 ini **selalu muncul**
bila kolom pelanggar dapat diekstrak; bila metadata tidak tersedia, field `details` tidak disertakan.

#### 500 — Internal Server Error

Terjadi ketika ada error database atau error sistem yang tidak tertangani.

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An error occurred while adding supplier data",
  "details": "detail error teknis (hanya di development mode)",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

---

## Field Otomatis

Primary key dan audit columns diisi secara otomatis oleh sistem. Audit columns dikelola secara otomatis (dikecualikan dari `fieldName` payload) dengan nama default `created_at`, `created_by`, `updated_at`, `updated_by`:

| Field | Nilai Otomatis | Kondisi |
|-------|---------------|---------|
| Primary key (e.g. `supplier_id`) | UUIDv7 | Jika tidak disertakan, kosong, atau bernilai `"-"` |
| `created_at` | SQL `CURRENT_TIMESTAMP` (waktu server) | Selalu ditetapkan saat create |
| `created_by` | user_id hasil resolusi berurutan: nilai di request body → header `x-user-id` → fallback `"system"` | Selalu ditetapkan saat create |
| `updated_at` | SQL `CURRENT_TIMESTAMP` (waktu server) | Ikut ditetapkan saat create |
| `updated_by` | Sama dengan resolusi `created_by` | Ikut ditetapkan saat create |

> **Catatan:** Nama keempat kolom dapat dikustomisasi atau dimatikan (`auditColumns = null`) melalui konfigurasi model. Nilai default mengikuti konvensi audit columns standar di atas.

---

## Perilaku Cache

| Aspek | Nilai |
|-------|-------|
| Di-cache | Tidak (write operation) |
| Invalidasi | Ya, seluruh cache endpoint ini (`list`, `datatables`, `lookup`) di-invalidasi setelah create berhasil |
| Cascade | Ya, cache processor yang bergantung pada tabel ini juga di-invalidasi |
| Key pattern | `rf:{project}:{endpoint}:*` |

---

## Event Lifecycle

Jika component engine dikonfigurasi, endpoint `/create` akan menjalankan event hooks dalam urutan berikut:

```
Request masuk
  → Validasi field (jika ada fieldValidation)
  → onBeforeInsert (pre-hook)
  → INSERT ke database (dalam transaction)
  → onAfterInsert (post-hook)
  → Invalidasi cache
  → Response 201
```

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| Field Validation | — | Aturan validasi deklaratif untuk setiap field |
| Hash (bcrypt) | — | Auto-hash field password sebelum insert |
| Cache | — | Invalidasi cache setelah operasi berhasil |
| Component Engine | — | Event lifecycle hooks (onBeforeInsert, onAfterInsert) |
| Body Options | — | Opsi tambahan via format `{data, options}` |
| Distributed Lock | — | Per-record locking (tidak berlaku untuk create) |
