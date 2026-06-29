# Change Status — Endpoint Perubahan Status Record

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/change-status` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `200 OK` |
| **Database** | PostgreSQL |
| **Parameter Wajib** | Primary key + `status` |
| **Operasi** | `UPDATE table SET status = $1 WHERE pk = $2 RETURNING *` |
| **Validasi Transisi** | Opsional via `workflow.transitions` |
| **Hook** | API call per target status (`onBefore`, `onAfter`) |
| **Hook Mode** | `blocking: true` (rollback jika gagal) atau `blocking: false` (fire-and-forget) |
| **Cache** | Invalidasi otomatis setelah change-status berhasil |
| **Distributed Lock** | Per-record WRITE lock (jika diaktifkan) |
| **Event Lifecycle** | `onBeforeWorkflow` → UPDATE → `onAfterWorkflow` |

---

## Ikhtisar

Endpoint `/change-status` digunakan untuk mengubah status record dalam satu operasi transaksional. Fitur ini mendukung validasi transisi (status A hanya boleh ke status B), dan hook API call yang dijalankan sebelum atau sesudah perubahan status.

Nama status bersifat fleksibel dan ditentukan oleh masing-masing project. Contoh di dokumen ini menggunakan `draft`, `confirmed`, `closed`, `cancelled` sesuai struktur database mini-inventory. Project lain bisa menggunakan nama status yang berbeda (misal: `approved`, `rejected`, `submitted`) selama sesuai dengan CHECK constraint di database.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/stock-inbound/change-status
```

---

## Format Request

### Parameter

| Parameter | Tipe | Wajib | Keterangan |
|-----------|------|:-----:|------------|
| `{primaryKey}` atau `id` | string | **Ya** | Nilai primary key record yang akan diubah statusnya |
| `status` | string | **Ya** | Status tujuan (target status) |
| `remarks` | string | Tidak | Catatan terkait perubahan status. Disimpan jika kolom `remarks` ada di `fieldName` |
| `updated_by` | string | Tidak | User yang melakukan perubahan. Disimpan jika kolom `updated_by` ada di `fieldName` |

### Contoh Request

**1. Confirm stock inbound:**
```json
{
  "stock_inbound_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "confirmed",
  "remarks": "Stock received and verified",
  "updated_by": "user-mgr-001"
}
```

**2. Close document:**
```json
{
  "stock_inbound_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "closed",
  "remarks": "Period closed"
}
```

**3. Cancel document (menggunakan alias `id`):**
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "cancelled",
  "remarks": "Supplier sent wrong items"
}
```

---

## Body Options (Opsional)

Endpoint `/change-status` mendukung format request body alternatif dengan wrapper `{data, options}`. Format ini memungkinkan pengiriman opsi tambahan yang dapat dibaca oleh component engine handler dan processor.

### Format dengan Options

```json
{
  "data": {
    "stock_inbound_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "status": "confirmed",
    "remarks": "Stock received and verified",
    "updated_by": "user-mgr-001"
  },
  "options": {
    "send_notification": true,
    "trigger_stock_update": true
  }
}
```

### Aturan

- Body harus memiliki **tepat 2 key**: `data` dan `options`
- `data` harus berupa object (berisi primary key + status tujuan)
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
  "message": "Status changed from draft to confirmed",
  "data": {
    "stock_inbound_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "inbound_number": "INB/2026/001",
    "inbound_date": "2026-04-01",
    "warehouse_id": "wh-001",
    "supplier_id": "sup-001",
    "status": "confirmed",
    "total_items": 2,
    "total_qty": 30,
    "total_amount": 170000000,
    "updated_at": "2026-04-03T10:00:00.000Z",
    "updated_by": "user-mgr-001"
  },
  "workflow": {
    "previousStatus": "draft",
    "newStatus": "confirmed",
    "hooksExecuted": 1
  },
  "timestamp": "2026-04-03T10:00:00.000Z"
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `success` | boolean | `true` jika perubahan status berhasil |
| `message` | string | `"Status changed from {old} to {new}"` |
| `data` | object | Record lengkap setelah status diubah (hasil `RETURNING *`) |
| `workflow.previousStatus` | string | Status sebelum perubahan |
| `workflow.newStatus` | string | Status setelah perubahan |
| `workflow.hooksExecuted` | number | Jumlah total hook yang dieksekusi (onBefore + onAfter) |
| `timestamp` | string | Waktu eksekusi (ISO 8601) |

### Response Error

#### 400 — Payload Kosong

```json
{
  "success": false,
  "error": "Invalid payload",
  "message": "Payload cannot be empty",
  "timestamp": "2026-04-03T10:00:00.000Z"
}
```

#### 400 — Primary Key Tidak Ada

```json
{
  "success": false,
  "error": "Missing required field",
  "message": "Primary key (stock_inbound_id) or id is required for change-status",
  "timestamp": "2026-04-03T10:00:00.000Z"
}
```

#### 400 — Status Tidak Ada

```json
{
  "success": false,
  "error": "Missing required field",
  "message": "status is required for change-status",
  "timestamp": "2026-04-03T10:00:00.000Z"
}
```

#### 400 — Status Field Tidak Valid

```json
{
  "success": false,
  "error": "Validation error",
  "message": "Status field \"custom_status\" is not a valid field in this table",
  "timestamp": "2026-04-03T10:00:00.000Z"
}
```

#### 404 — Data Tidak Ditemukan

```json
{
  "success": false,
  "error": "Data not found",
  "message": "stock-inbound data not found",
  "timestamp": "2026-04-03T10:00:00.000Z"
}
```

#### 409 — Resource Busy (Lock)

Terjadi ketika record sedang diproses oleh request lain.

```json
{
  "success": false,
  "error": "Resource busy",
  "message": "Failed to acquire lock for change-status operation. The resource is currently busy, please try again later.",
  "timestamp": "2026-04-03T10:00:00.000Z"
}
```

#### 409 — Concurrent Modification

Terjadi ketika status record diubah oleh request lain saat operasi berlangsung (optimistic concurrency check via status guard clause pada UPDATE).

```json
{
  "success": false,
  "error": "Concurrent modification",
  "message": "...",
  "timestamp": "2026-04-03T10:00:00.000Z"
}
```

#### 422 — Transisi Status Tidak Diizinkan

Terjadi ketika `workflow.transitions` dikonfigurasi dan perpindahan status tidak terdaftar.

```json
{
  "success": false,
  "error": "Status transition not allowed",
  "message": "Cannot change status from 'draft' to 'closed'. Allowed transitions: confirmed, cancelled",
  "timestamp": "2026-04-03T10:00:00.000Z"
}
```

#### 422 — Tidak Ada Transisi untuk Status Saat Ini

Terjadi ketika status saat ini tidak memiliki entry di `workflow.transitions`.

```json
{
  "success": false,
  "error": "Status transition not allowed",
  "message": "No transitions defined for current status: 'unknown_status'",
  "timestamp": "2026-04-03T10:00:00.000Z"
}
```

#### 502 — Hook API Call Gagal (Blocking)

Terjadi ketika hook dengan `blocking: true` mengembalikan error.

```json
{
  "success": false,
  "error": "Workflow hook failed",
  "message": "Workflow onAfter hook failed: Blocking hook failed: HTTP 500: Internal Server Error",
  "timestamp": "2026-04-03T10:00:00.000Z"
}
```

#### 500 — Internal Server Error

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An error occurred while changing status",
  "details": "detail error teknis (hanya di development mode)",
  "timestamp": "2026-04-03T10:00:00.000Z"
}
```

---

## Validasi Transisi

Validasi transisi bersifat **opsional**. Jika `workflow.transitions` dikonfigurasi, hanya perpindahan status yang terdaftar yang diizinkan. Jika tidak dikonfigurasi, status bisa berubah ke nilai apa pun.

**Contoh konfigurasi transitions:**
```json
{
  "transitions": {
    "draft": ["confirmed", "cancelled"],
    "confirmed": ["closed", "cancelled"],
    "closed": [],
    "cancelled": []
  }
}
```

| Konfigurasi | Perilaku |
|-------------|----------|
| `"draft": ["confirmed", "cancelled"]` | Dari `draft` bisa ke `confirmed` atau `cancelled` |
| `"closed": []` | Terminal state, tidak bisa pindah ke status mana pun |
| `transitions` tidak ada / `null` | Bebas pindah ke status mana pun tanpa validasi |

**Diagram transisi** (berdasarkan contoh di atas):

```
                          cancel
              ┌──────────────────────────────┐
              │                              ▼
┌─────────┐  confirm  ┌───────────┐    ┌───────────┐
│  DRAFT  │ ────────► │ CONFIRMED │    │ CANCELLED │
└─────────┘           └─────┬──┬──┘    └───────────┘
                            │  │        (terminal)
                      close │  │ cancel     ▲
                            ▼  └────────────┘
                      ┌──────────┐
                      │  CLOSED  │
                      └──────────┘
                       (terminal)
```

---

## Hook API Call

Hook memungkinkan pemanggilan API ke endpoint lain saat status berubah. Hook dikonfigurasi **per target status** di `workflow.hooks`.

### Timing Eksekusi

| Timing | Kapan Dieksekusi | Jika Blocking Hook Gagal |
|--------|------------------|--------------------------|
| `onBefore` | Sebelum `UPDATE` SQL | ROLLBACK, status tidak berubah |
| `onAfter` | Setelah `UPDATE` SQL, sebelum `COMMIT` | ROLLBACK, status dikembalikan |

### Properti Hook

| Property | Tipe | Default | Keterangan |
|----------|------|---------|------------|
| `type` | string | - | Tipe hook. Saat ini hanya `"api"` |
| `method` | string | `"POST"` | HTTP method: `POST`, `PUT`, `PATCH` |
| `url` | string | - | URL endpoint tujuan. Mendukung 3 format penulisan |
| `body` | object | `{}` | Request body yang dikirim ke endpoint tujuan. Seluruh isi `body` sepenuhnya menjadi tanggung jawab endpoint penerima. Workflow engine hanya mengirimkan body apa adanya tanpa memperlakukan field mana pun secara khusus |
| `headers` | object | `{}` | Custom HTTP headers |
| `blocking` | boolean | `false` | `true`: rollback jika gagal. `false`: fire-and-forget |
| `timeout` | number | `10000` | Timeout dalam milidetik |

### Format URL

| Format | Deteksi | Resolusi |
|--------|---------|----------|
| **Full URL** | Dimulai `http://` atau `https://` | Digunakan apa adanya |
| **Absolute path** | Dimulai `/api/` | Ditambah `http://{SERVER_ADDRESS}:{PORT}` |
| **Short path** | Dimulai `/` tanpa `/api/` | Ditambah `http://{SERVER_ADDRESS}:{PORT}/api/{project}` |

**Contoh — 3 penulisan menghasilkan URL yang identik:**

```
Full URL      : "http://localhost:3000/api/mini-inventory/stock-management/process-inbound"
Absolute path : "/api/mini-inventory/stock-management/process-inbound"
Short path    : "/stock-management/process-inbound"
```

### Template Variable dalam Hook

| Variable | Keterangan |
|----------|------------|
| `{{id}}` | Nilai primary key record |
| `{{newStatus}}` | Status tujuan |
| `{{oldStatus}}` | Status sebelum perubahan |
| `{{remarks}}` | Catatan dari request body |
| `{{tableName}}` | Nama tabel |
| `{{user_id}}` | User yang melakukan perubahan |
| `{{timestamp}}` | Waktu perubahan (ISO 8601) |
| `{{record.<field>}}` | Field dari data record di database. Pada `onBefore`: data sebelum perubahan. Pada `onAfter`: data setelah perubahan |
| `{{requestData.<field>}}` | Field dari request body yang dikirim client |

### Contoh Konfigurasi Hook

```json
{
  "hooks": {
    "confirmed": {
      "onAfter": [
        {
          "type": "api",
          "method": "POST",
          "url": "/stock-management/process-inbound",
          "body": {
            "stock_inbound_id": "{{id}}",
            "inbound_number": "{{record.inbound_number}}",
            "status": "{{newStatus}}",
            "previous_status": "{{oldStatus}}",
            "action": "confirm-inbound"
          },
          "blocking": true,
          "timeout": 15000
        }
      ]
    },
    "cancelled": {
      "onAfter": [
        {
          "type": "api",
          "method": "POST",
          "url": "/stock-management/reverse-inbound",
          "body": {
            "stock_inbound_id": "{{id}}",
            "action": "cancel-inbound"
          },
          "blocking": false,
          "timeout": 10000
        }
      ]
    }
  }
}
```

---

## Konfigurasi Payload

Endpoint `/change-status` memerlukan `action.workflow = true` dan konfigurasi `workflow` di payload JSON:

```json
{
  "action": { "workflow": true },
  "workflow": {
    "statusField": "status",
    "transitions": {
      "draft": ["confirmed", "cancelled"],
      "confirmed": ["closed", "cancelled"],
      "closed": [],
      "cancelled": []
    },
    "hooks": {
      "confirmed": {
        "onAfter": [
          {
            "type": "api",
            "method": "POST",
            "url": "/stock-management/process-inbound",
            "body": {
              "stock_inbound_id": "{{id}}",
              "action": "confirm-inbound"
            },
            "blocking": true
          }
        ]
      }
    }
  }
}
```

| Property | Tipe | Default | Keterangan |
|----------|------|---------|------------|
| `statusField` | string | `"status"` | Nama kolom database yang menyimpan status. Harus terdaftar di `fieldName` |
| `transitions` | object | `null` | Map transisi yang diizinkan. `null` = tanpa validasi |
| `hooks` | object | `{}` | Map hook per target status |

---

## Alur Eksekusi

```
1.  Acquire write lock (jika distributed lock aktif)
2.  BEGIN TRANSACTION
3.  SELECT * FROM table WHERE pk = id → oldData
4.  Validate transition (jika transitions dikonfigurasi)
5.  Component Engine: onBeforeWorkflow (jika components dikonfigurasi)
6.  Workflow hooks: onBefore (API calls per target status)
7.  UPDATE table SET status = $1, updated_at = NOW() WHERE pk = $2 RETURNING *
8.  Workflow hooks: onAfter (API calls per target status)
9.  Component Engine: onAfterWorkflow (jika components dikonfigurasi)
10. COMMIT
11. Release write lock
12. Invalidate cache
13. Return response
```

Seluruh langkah 2 sampai 10 berjalan dalam satu database transaction. Kegagalan di langkah mana pun (termasuk blocking hook) menyebabkan ROLLBACK.

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| Workflow Basic (Deep Dive) | — | Dokumentasi lengkap termasuk prasyarat, skenario, dan troubleshooting |
| Distributed Lock | — | Per-record WRITE lock saat change-status |
| Component Engine | — | Event hooks (onBeforeWorkflow, onAfterWorkflow) |
| Body Options | — | Opsi tambahan via format `{data, options}` |
