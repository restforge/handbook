# Update — Endpoint Pembaruan Data

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/update` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `200 OK` |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Validasi** | `fieldValidation` dari payload JSON (opsional) |
| **Partial Update** | Didukung, hanya field yang dikirim yang diubah |
| **Cache** | Invalidasi otomatis setelah update berhasil |
| **Distributed Lock** | Per-record WRITE lock (jika diaktifkan) |
| **Event Lifecycle** | `onBeforeUpdate` → UPDATE → `onAfterUpdate` |

---

## Ikhtisar

Endpoint `/update` digunakan untuk memperbarui satu record existing di database. Request body **wajib** menyertakan primary key untuk mengidentifikasi record yang akan diubah. Hanya field yang disertakan dalam request yang akan diperbarui (partial update). Field yang tidak dikirim tetap tidak berubah.

Jika distributed lock diaktifkan, sistem akan memperoleh per-record WRITE lock sebelum memulai operasi update dan melepaskannya setelah selesai (baik sukses maupun gagal). Hal ini mencegah konflik ketika dua request mencoba mengubah record yang sama secara bersamaan.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/supplier/update
POST http://localhost:3000/api/mini-inventory/item-product/update
```

---

## Format Request

### Parameter

Request body berisi JSON object dengan primary key (wajib) dan field-value yang akan diperbarui.

| Parameter | Tipe | Wajib | Keterangan |
|-----------|------|:-----:|------------|
| `{primaryKey}` | string | **Ya** | Primary key record yang akan diubah (e.g. `supplier_id`) |
| `{field}` | any | Tidak | Field-value yang akan diperbarui. Hanya field di `fieldName` payload yang diproses |

### Contoh Request

**1. Update beberapa field:**
```json
{
  "supplier_id": "550e8400-e29b-41d4-a716-446655440000",
  "supplier_name": "PT Maju Jaya Baru",
  "email": "new-contact@majujaya.com",
  "phone": "021-5559999"
}
```

**2. Partial update — hanya satu field:**
```json
{
  "supplier_id": "550e8400-e29b-41d4-a716-446655440000",
  "is_active": false
}
```

**3. Update field datetime:**
```json
{
  "contract_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "end_date": "2027-12-31",
  "notes": "Perpanjangan kontrak 1 tahun"
}
```

---

## Body Options (Opsional)

Endpoint `/update` mendukung format request body alternatif dengan wrapper `{data, options}`. Format ini memungkinkan pengiriman opsi tambahan yang dapat dibaca oleh component engine handler dan processor.

### Format dengan Options

```json
{
  "data": {
    "supplier_id": "550e8400-e29b-41d4-a716-446655440000",
    "supplier_name": "PT Maju Jaya Baru",
    "email": "new-contact@majujaya.com"
  },
  "options": {
    "skip_audit_log": true,
    "notify_admin": false
  }
}
```

### Aturan

- Body harus memiliki **tepat 2 key**: `data` dan `options`
- `data` harus berupa object (berisi primary key + field yang akan diubah)
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
  "message": "supplier data successfully updated",
  "data": {
    "supplier_id": "550e8400-e29b-41d4-a716-446655440000",
    "supplier_code": "SUP-001",
    "supplier_name": "PT Maju Jaya Baru",
    "email": "new-contact@majujaya.com",
    "phone": "021-5559999",
    "address": "Jl. Industri No. 10",
    "is_active": true,
    "created_at": "2026-03-30T10:30:00.000Z",
    "created_by": "system",
    "updated_at": "2026-03-30T14:45:00.000Z",
    "updated_by": "system"
  },
  "timestamp": "2026-03-30T14:45:00.000Z"
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `success` | boolean | `true` jika update berhasil |
| `message` | string | `"{endpoint} data successfully updated"` |
| `data` | object | Record lengkap setelah diperbarui (seluruh field, bukan hanya yang diubah) |
| `timestamp` | string | Waktu eksekusi (ISO 8601) |

### Response Error

#### 400 — Payload Kosong

```json
{
  "success": false,
  "error": "Invalid payload",
  "message": "Payload cannot be empty",
  "timestamp": "2026-03-30T14:45:00.000Z"
}
```

#### 400 — Primary Key Tidak Ada

Terjadi ketika request body tidak menyertakan primary key.

```json
{
  "success": false,
  "error": "Missing required field",
  "message": "Primary key (supplier_id) is required for update",
  "timestamp": "2026-03-30T14:45:00.000Z"
}
```

#### 400 — Validasi Gagal

```json
{
  "success": false,
  "error": "Validation failed",
  "message": "Invalid data",
  "errors": {
    "email": ["Field email must be a valid email address"],
    "supplier_name": ["Field supplier_name must be at most 100 characters"]
  },
  "timestamp": "2026-03-30T14:45:00.000Z"
}
```

#### 500 — Resource Sedang Dikunci (Distributed Lock)

Terjadi ketika record sedang diproses oleh request lain dan distributed lock aktif. Kegagalan memperoleh lock ditangani sebagai error 500 (catch-all).

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An error occurred while updating supplier data",
  "details": "Failed to acquire lock for update operation. The resource is currently busy, please try again later.",
  "timestamp": "2026-03-30T14:45:00.000Z"
}
```

> **Catatan:** Field `details` hanya muncul di development mode. Di production, pesan lock error tidak ditampilkan ke client.

#### 404 — Data Tidak Ditemukan

Terjadi ketika primary key yang diberikan tidak cocok dengan record mana pun, atau update menghasilkan 0 rows affected.

```json
{
  "success": false,
  "error": "Data not found",
  "message": "supplier data not found",
  "timestamp": "2026-03-30T14:45:00.000Z"
}
```

#### 409 — Duplicate Entry

Terjadi ketika perubahan melanggar unique constraint (PostgreSQL error code `23505`).

```json
{
  "success": false,
  "error": "Duplicate entry",
  "message": "A record with this value already exists",
  "details": {
    "email": ["A record with this value already exists"]
  },
  "timestamp": "2026-03-30T14:45:00.000Z"
}
```

Field `details` adalah map `{ "<kolom>": ["<pesan>"] }` yang menunjuk kolom pelanggar
unique constraint, berguna untuk memetakan error ke field form. Untuk unique constraint
multi-kolom, setiap kolom dipetakan ke pesan
`"A record with this combination of values already exists"`. Berbeda dengan `details`
pada 500 (string, hanya `NODE_ENV=development`), `details` pada 409 ini **selalu muncul**
bila kolom pelanggar dapat diekstrak; bila metadata tidak tersedia, field `details` tidak disertakan.

#### 500 — Internal Server Error

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An error occurred while updating supplier data",
  "details": "detail error teknis (hanya di development mode)",
  "timestamp": "2026-03-30T14:45:00.000Z"
}
```

---

## Field Otomatis

Audit columns dikelola secara otomatis dengan nama default `updated_at` dan `updated_by`. Operasi update hanya menyentuh dua kolom ini:

| Field | Nilai Otomatis | Kondisi |
|-------|---------------|---------|
| `updated_at` | SQL `CURRENT_TIMESTAMP` (waktu server) | Selalu ditetapkan saat update |
| `updated_by` | user_id hasil resolusi berurutan: nilai di request body → header `x-user-id` → fallback `"system"` | Selalu ditetapkan saat update |

> **Catatan:** Field `created_at` dan `created_by` **tidak** diubah oleh operasi update. Nama kolom dan nilai mengikuti konvensi audit columns default. Kolom legacy `modified_by`/`modified_date` tidak lagi ditangani runtime.

---

## Perilaku Cache

| Aspek | Nilai |
|-------|-------|
| Di-cache | Tidak (write operation) |
| Invalidasi | Ya, seluruh cache endpoint ini (`list`, `datatables`, `lookup`) di-invalidasi setelah update berhasil |
| Cascade | Ya, cache processor yang bergantung pada tabel ini juga di-invalidasi |
| Key pattern | `rf:{project}:{endpoint}:*` |

---

## Distributed Lock

Jika distributed lock diaktifkan (`LOCK_ENABLED=true`), operasi update akan:

1. Memperoleh WRITE lock untuk record spesifik (`{project}:{endpoint}:{recordId}`)
2. Menjalankan operasi UPDATE
3. Melepaskan lock di akhir operasi (baik sukses maupun error)

Jika lock gagal diperoleh (record sedang diproses request lain), kegagalan tersebut ditangani sebagai error **500** (lihat bagian "500 — Resource Sedang Dikunci" di atas).

---

## Event Lifecycle

```
Request masuk
  → Validasi field (jika ada fieldValidation)
  → Acquire WRITE lock (jika distributed lock aktif)
  → Fetch old data (untuk event context)
  → onBeforeUpdate (pre-hook)
  → UPDATE ke database (dalam transaction)
  → onAfterUpdate (post-hook)
  → Invalidasi cache
  → Release lock
  → Response 200
```

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| Field Validation | — | Aturan validasi deklaratif untuk setiap field |
| Hash (bcrypt) | — | Auto-hash field password sebelum update |
| Cache | — | Invalidasi cache setelah operasi berhasil |
| Component Engine | — | Event lifecycle hooks (onBeforeUpdate, onAfterUpdate) |
| Body Options | — | Opsi tambahan via format `{data, options}` |
| Distributed Lock | — | Per-record WRITE locking untuk mencegah race condition |
