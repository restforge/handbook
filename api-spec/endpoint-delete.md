# Delete — Endpoint Penghapusan Data

---

## Referensi Cepat (Quick Reference)

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/delete` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `200 OK` |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Parameter Wajib** | `where` (menentukan record yang akan dihapus) |
| **Cache** | Invalidasi otomatis setelah delete berhasil |
| **Distributed Lock** | Per-record WRITE lock (jika diaktifkan) |
| **Event Lifecycle** | `onBeforeDelete` → DELETE → `onAfterDelete` |

---

## Ikhtisar (Overview)

Endpoint `/delete` digunakan untuk menghapus satu atau lebih record dari tabel database. Parameter `where` bersifat **wajib** sehingga request tanpa parameter `where` akan langsung ditolak dengan error 400. Hal ini mencegah penghapusan data secara tidak sengaja.

Sebelum menghapus, sistem akan mengambil data lama (old data) untuk keperluan event lifecycle dan audit. Data yang berhasil dihapus dikembalikan dalam response `deleted_items`.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/supplier/delete
POST http://localhost:3000/api/mini-inventory/item-product/delete
```

---

## Format Request (Request Format)

### Parameter (Parameters)

| Parameter | Tipe | Wajib | Keterangan |
|-----------|------|:-----:|------------|
| `where` | array / object | **Ya** | Kondisi untuk menentukan record yang akan dihapus. Lihat [Format WHERE](README.md#format-where-clause) |

### Contoh Request (Request Examples)

**1. Delete by primary key (format sederhana):**
```json
{
  "where": [
    { "key": "supplier_id", "value": "550e8400-e29b-41d4-a716-446655440000" }
  ]
}
```

**2. Delete dengan multiple conditions:**
```json
{
  "where": [
    { "key": "company_id", "value": "company-uuid" },
    { "key": "is_active", "value": "false" }
  ]
}
```

**3. Delete dengan format kompleks:**
```json
{
  "where": {
    "logic": "AND",
    "conditions": [
      { "key": "status", "operator": "=", "value": "deleted" },
      { "key": "updated_at", "operator": "<", "value": "2025-01-01" }
    ]
  }
}
```

---

## Body Options (Opsional)

Endpoint `/delete` mendukung format request body alternatif dengan wrapper `{data, options}`. Format ini memungkinkan pengiriman opsi tambahan yang dapat dibaca oleh component engine handler dan processor.

### Format dengan Options

```json
{
  "data": {
    "where": [
      { "key": "supplier_id", "value": "550e8400-e29b-41d4-a716-446655440000" }
    ]
  },
  "options": {
    "soft_delete": true,
    "reason": "Data tidak lagi relevan"
  }
}
```

### Aturan (Rules)

- Body harus memiliki **tepat 2 key**: `data` dan `options`
- `data` harus berupa object (berisi parameter `where` untuk menentukan record yang dihapus)
- `options` harus berupa object (berisi flag/parameter tambahan)
- Jika format ini tidak terdeteksi, body diproses sebagai format standar (backward compatible)
- Nilai `options` tersedia di component handler sebagai template variable `{options}` dan di processor melalui `req.bodyOptions`

---

## Format Response (Response Format)

### Response Sukses (Success Response)

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "message": "Successfully deleted 1 record",
  "deleted_count": 1,
  "deleted_items": [
    {
      "supplier_id": "550e8400-e29b-41d4-a716-446655440000",
      "supplier_code": "SUP-001",
      "supplier_name": "PT Maju Jaya",
      "email": "contact@majujaya.com",
      "is_active": false,
      "created_at": "2026-03-30T10:30:00.000Z",
      "created_by": "system",
      "updated_at": "2026-03-30T10:30:00.000Z",
      "updated_by": "system"
    }
  ],
  "timestamp": "2026-03-30T15:00:00.000Z"
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `success` | boolean | `true` jika operasi berhasil |
| `message` | string | `"Successfully deleted {n} record(s)"` atau `"No records deleted"` |
| `deleted_count` | number | Jumlah record yang berhasil dihapus |
| `deleted_items` | array | Array berisi data record yang dihapus (dari `RETURNING *`) |
| `timestamp` | string | Waktu eksekusi (ISO 8601) |

### Response Error (Error Response)

#### 400 — Payload Kosong

```json
{
  "success": false,
  "error": "Invalid payload",
  "message": "Payload cannot be empty",
  "timestamp": "2026-03-30T15:00:00.000Z"
}
```

#### 400 — Parameter WHERE Tidak Ada

```json
{
  "success": false,
  "error": "Missing required field",
  "message": "Invalid request format: where parameter is required",
  "example": {
    "where": [{ "key": "id", "value": "your-id-value" }]
  },
  "timestamp": "2026-03-30T15:00:00.000Z"
}
```

#### 400 — Format WHERE Tidak Valid

Terjadi ketika struktur `where` tidak sesuai format yang didukung.

```json
{
  "success": false,
  "error": "Invalid where format",
  "message": "Invalid where format",
  "example": {
    "where": [{ "key": "id", "value": "your-id-value" }]
  },
  "timestamp": "2026-03-30T15:00:00.000Z"
}
```

#### 409 — Foreign Key Constraint

Terjadi ketika record yang akan dihapus masih direferensikan oleh data di tabel lain (PostgreSQL error code `23503`). Pada endpoint delete, FK violation di-map ke 409 (bukan 400 seperti di create/update) karena ini merupakan konflik state, yaitu record tidak bisa dihapus selama masih digunakan.

```json
{
  "success": false,
  "error": "Foreign key constraint",
  "message": "Cannot delete: record is still referenced by other data",
  "timestamp": "2026-03-30T15:00:00.000Z"
}
```

#### 500 — Resource Sedang Dikunci (Distributed Lock)

Terjadi ketika record sedang diproses oleh request lain. Kegagalan akuisisi lock ditangani sebagai error 500 (catch-all).

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An error occurred while deleting supplier data",
  "details": "Failed to acquire lock for delete operation. The resource is currently busy, please try again later.",
  "timestamp": "2026-03-30T15:00:00.000Z"
}
```

> **Catatan:** Field `details` hanya muncul di development mode.

#### 404 — Data Tidak Ditemukan

Terjadi ketika tidak ada record yang cocok dengan kondisi WHERE.

```json
{
  "success": false,
  "error": "Data not found",
  "message": "supplier data not found",
  "timestamp": "2026-03-30T15:00:00.000Z"
}
```

#### 500 — Internal Server Error

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An error occurred while deleting supplier data",
  "details": "detail error teknis (hanya di development mode)",
  "timestamp": "2026-03-30T15:00:00.000Z"
}
```

---

## Perilaku Cache (Cache Behavior)

| Aspek | Nilai |
|-------|-------|
| Di-cache | Tidak (write operation) |
| Invalidasi | Ya, seluruh cache endpoint ini di-invalidasi **hanya jika** ada record yang berhasil dihapus (`deleted_count > 0`) |
| Cascade | Ya, cache processor yang bergantung pada tabel ini juga di-invalidasi |
| Key pattern | `rf:{project}:{endpoint}:*` |

---

## Distributed Lock

Jika distributed lock diaktifkan, operasi delete akan:

1. Mengekstrak record ID dari parameter `where` (jika dapat diidentifikasi)
2. Mengakuisisi WRITE lock untuk record spesifik
3. Mengambil old data sebelum penghapusan (untuk event context)
4. Menjalankan operasi DELETE
5. Melepaskan lock di akhir operasi

> **Catatan:** Lock hanya diakuisisi jika record ID dapat diekstrak dari kondisi WHERE. Jika WHERE menggunakan kondisi kompleks yang tidak menyertakan primary key, operasi akan berjalan tanpa lock.

---

## Event Lifecycle

```
Request masuk
  → Validasi parameter WHERE
  → Acquire WRITE lock (jika distributed lock aktif)
  → Fetch old data (untuk event context)
  → onBeforeDelete (pre-hook)
  → DELETE dari database (dalam transaction, dengan RETURNING)
  → onAfterDelete (post-hook)
  → Invalidasi cache (jika ada data yang dihapus)
  → Release lock
  → Response 200
```

---

## Fitur Terkait (Related Features)

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| Cache | — | Invalidasi cache setelah operasi berhasil |
| Component Engine | — | Event lifecycle hooks (onBeforeDelete, onAfterDelete) |
| Body Options | — | Opsi tambahan via format `{data, options}` |
| Distributed Lock | — | Per-record WRITE locking untuk mencegah race condition |
