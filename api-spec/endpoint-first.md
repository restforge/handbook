# First — Endpoint Pengambilan Record Tunggal

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/first` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `200 OK` |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Parameter Wajib** | `where` (kondisi sederhana `{ key, value }`) |
| **Operator** | Hanya `=` (equality) |
| **Hasil** | Maksimum 1 record (`LIMIT 1`) |
| **Cache** | **Tidak** diterapkan, data yang dibaca selalu dari database |
| **Default Scope** | **Tidak** diterapkan |

---

## Ikhtisar

Endpoint `/first` dirancang untuk mengambil satu record dari database berdasarkan kondisi sederhana (equality). Endpoint ini digunakan untuk keperluan **View/Edit form** di frontend, yaitu saat pengguna membuka detail satu record.

Berbeda dengan `/read`, endpoint `/first` hanya mendukung satu kondisi WHERE dengan operator `=`, tidak mendukung format WHERE kompleks (`conditions`, `logic`, operator `LIKE`/`BETWEEN`/`IN`), dan selalu mengembalikan maksimum 1 record.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/supplier/first
POST http://localhost:3000/api/mini-inventory/item-product/first
```

---

## Format Request

### Parameter

| Parameter | Tipe | Wajib | Keterangan |
|-----------|------|:-----:|------------|
| `where` | object / array | **Ya** | Kondisi pencarian sederhana. Hanya mendukung operator `=` |
| `select` | array | Tidak | Kolom spesifik yang dikembalikan. Default: seluruh kolom (`*`) |

### Format WHERE

**Format object (direkomendasikan):**
```json
{
  "where": { "key": "supplier_id", "value": "550e8400-e29b-41d4-a716-446655440000" }
}
```

**Format array (kompatibilitas mundur, maksimum 1 elemen):**
```json
{
  "where": [{ "key": "supplier_id", "value": "550e8400-e29b-41d4-a716-446655440000" }]
}
```

> **Format yang TIDAK didukung:** `conditions`, `logic`, operator selain `=`. Gunakan endpoint `/read` untuk query kompleks.

### Contoh Request

**1. Ambil record berdasarkan primary key:**
```json
{
  "where": { "key": "supplier_id", "value": "550e8400-e29b-41d4-a716-446655440000" }
}
```

**2. Dengan kolom selektif:**
```json
{
  "where": { "key": "supplier_id", "value": "550e8400-e29b-41d4-a716-446655440000" },
  "select": ["supplier_id", "supplier_name", "email", "phone"]
}
```

**3. Ambil berdasarkan kolom unik (bukan PK):**
```json
{
  "where": { "key": "supplier_code", "value": "SUP-001" }
}
```

---

## Format Response

### Response Sukses — Data Ditemukan

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "count": 1,
  "data": [
    {
      "supplier_id": "550e8400-e29b-41d4-a716-446655440000",
      "supplier_code": "SUP-001",
      "supplier_name": "PT Maju Jaya",
      "email": "contact@majujaya.com",
      "phone": "021-5551234",
      "is_active": true
    }
  ],
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

### Response Sukses — Data Tidak Ditemukan

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "count": 0,
  "data": [],
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

> **Catatan:** Data tidak ditemukan bukan merupakan error. Response tetap `200 OK` dengan `data` kosong.

| Field | Tipe | Keterangan |
|-------|------|------------|
| `success` | boolean | `true` jika query berhasil (terlepas dari ada/tidaknya data) |
| `count` | number | `1` jika ditemukan, `0` jika tidak |
| `data` | array | Array berisi 0 atau 1 record |
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

#### 400 — WHERE Tidak Ada

```json
{
  "success": false,
  "error": "Missing required field",
  "message": "Property where is required as {key, value} object",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — WHERE Key/Value Kosong

```json
{
  "success": false,
  "error": "Invalid where format",
  "message": "Where key and value are required",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Format WHERE Kompleks (Tidak Didukung)

```json
{
  "success": false,
  "error": "Invalid where format",
  "message": "Advanced where format is not supported in /first endpoint. Use /read endpoint for complex queries",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Field WHERE Tidak Valid

```json
{
  "success": false,
  "error": "Invalid where field",
  "message": "Invalid field: field_tidak_ada",
  "validFields": ["supplier_id", "supplier_code", "supplier_name"],
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Field Select Tidak Valid

```json
{
  "success": false,
  "error": "Invalid select fields",
  "message": "Invalid field(s): field_tidak_ada",
  "validFields": ["supplier_id", "supplier_code", "supplier_name"],
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

> Field `validFields` dalam respons error hanya disertakan saat `NODE_ENV=development`. Di produksi, field ini tidak ditampilkan (anti-disclosure).

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

## Proyeksi Whitelist

Kolom yang dapat direferensikan di `where.key` dan `select` dibatasi oleh **`readableFields`** — proyeksi yang identik dengan `/read` karena kedua endpoint berbagi sumber resolusi. Referensi kolom di luar `readableFields` ditolak 400 secara eksplisit (`"Invalid field: <nama_kolom>"`), bukan diabaikan. Bila soft-delete aktif, kolom soft-delete otomatis ditambahkan ke `readableFields`.

## Sumber Data

Resolusi sumber data **identik** dengan endpoint `/read`:

| Prioritas | Properti Payload | Keterangan |
|:---------:|------------------|------------|
| 1 | `viewName` | Jika didefinisikan dan berbeda dari `tableName` |
| 2 | `viewQuery` | Dibungkus sebagai subquery dengan alias `get_q` |
| 3 | `tableName` | Fallback terakhir |

---

## Perbedaan dengan Endpoint /read

| Aspek | `/first` | `/read` |
|-------|---------|--------|
| WHERE | Hanya 1 kondisi, operator `=` | Multiple conditions, semua operator |
| Logic (AND/OR) | Tidak didukung | Didukung |
| Hasil | Maks 1 record (`LIMIT 1`) | 1 — 5000 record (configurable) |
| Paginasi | Tidak | Ya |
| Cache | Tidak | Ya |
| Default Scope | Tidak | Ya (jika dikonfigurasi) |
| Kegunaan | View/Edit form detail | List browsing, query kompleks |

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| First (Deep Dive) | — | Dokumentasi lengkap endpoint first |
| Query Declarative | — | Konfigurasi viewName, viewQuery, dan sumber data |
