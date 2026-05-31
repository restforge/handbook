# Resource Lock (Manual Lock API)

> Penguncian eksplisit satu atau beberapa record sekaligus melalui REST API.

Kembali ke [ikhtisar Distributed Lock](README.md).

---

Resource Lock adalah mekanisme penguncian eksplisit untuk mengunci satu atau beberapa record sebelum melakukan serangkaian operasi. Lock di-acquire dan di-release secara manual melalui API, berbeda dengan per-record lock yang berjalan otomatis pada CRUD standar.

## Perbedaan dengan Per-Record Lock

| Aspek | Per-Record Lock | Resource Lock |
|-------|-----------------|---------------|
| Cara kerja | Otomatis di dalam operasi update/delete | Manual via API |
| Cakupan | 1 record per operasi | Banyak record sekaligus |
| Durasi | Selama operasi CRUD berlangsung | Selama caller membutuhkan (hingga TTL habis) |
| Endpoint | Tidak ada (berjalan di background) | `POST /lock/acquire`, `/release`, `/status` |

## Kapan Menggunakan Resource Lock

Gunakan resource lock apabila:

- Perlu mengunci beberapa record sekaligus sebelum memproses transaksi composite
- Terdapat validasi stok atau kuota yang harus konsisten di antara proses concurrent
- Dua atau lebih service dapat memodifikasi resource yang sama secara bersamaan

Tidak perlu resource lock apabila operasi hanya CRUD standar pada 1 record (per-record lock sudah mencukupi), operasi bersifat read-only, atau tidak ada risiko concurrent modification.

## Prasyarat

| Komponen | Keterangan |
|----------|------------|
| Redis Server | Redis harus aktif dan dapat diakses |
| `LOCK_DISTRIBUTED_ENABLED=true` | Wajib di-set agar endpoint resource lock terdaftar |

---

## Alur Dasar (Basic Flow)

Penggunaan resource lock mengikuti pola **acquire → operasi → release**. Release wajib ditempatkan di `finally` block agar selalu terpanggil meskipun terjadi error.

```javascript
const axios = require('axios');
const BASE_URL = 'http://localhost:3080/api/inventory';

async function processStockOutbound(items) {
  let lockToken = null;

  try {
    // 1. Acquire lock
    const lock = await axios.post(`${BASE_URL}/lock/acquire`, {
      resource: 'item_products',
      lock_keys: items.map(item => ({ key: 'item_product_id', value: item.item_product_id })),
      ttl: 30,
      owner: `checkout-${process.pid}`
    });
    if (!lock.data.success) throw new Error(`Lock gagal: ${lock.data.error}`);
    lockToken = lock.data.data.lock_token;

    // 2. Operasi bisnis
    const result = await axios.post(`${BASE_URL}/stock-outbound/create-composite`, {
      header: { outbound_date: '2026-02-25', warehouse_id: 'WH-001' },
      details: items
    });
    return result.data;

  } finally {
    // 3. Release lock (wajib di finally block)
    if (lockToken) {
      await axios.post(`${BASE_URL}/lock/release`, {
        resource: 'item_products',
        lock_keys: items.map(item => ({ key: 'item_product_id', value: item.item_product_id })),
        lock_token: lockToken
      });
    }
  }
}
```

---

## Referensi API (API Reference)

Endpoint resource lock berada di level project, bukan per-endpoint.

### POST /api/{project}/lock/acquire

Mengunci satu atau lebih record. Apabila salah satu record gagal dikunci, **seluruh permintaan dibatalkan** (all-or-nothing) sehingga tidak ada partial lock yang tersisa.

| Field | Tipe | Wajib | Keterangan |
|-------|------|-------|------------|
| `resource` | string | Ya | Nama resource yang dikunci (contoh: `"item_products"`) |
| `lock_keys` | array | Ya | Daftar record. Minimum 1, maksimum 100 item |
| `lock_keys[].key` | string | Ya | Nama kolom identifier (contoh: `"item_product_id"`) |
| `lock_keys[].value` | string | Ya | Nilai identifier record |
| `ttl` | number | Ya | Durasi lock dalam detik. Minimum 1, maksimum `LOCK_RESOURCE_MAX_TTL` |
| `owner` | string | Ya | Identifier pemanggil untuk keperluan trace pemegang lock |

Response sukses (HTTP 200):

```json
{
  "success": true,
  "data": {
    "lock_token": "service-checkout-d4e5:a1b2c3d4-...:1740000000000",
    "resource": "item_products",
    "locked_keys": [
      { "key": "item_product_id", "value": "101", "lock_key": "rf:rlock:item_products:item_product_id:101" }
    ],
    "ttl": 30,
    "owner": "service-checkout-d4e5",
    "acquired_at": "2026-02-25T10:00:00.000Z"
  }
}
```

Simpan nilai `lock_token` dari response, karena diperlukan untuk release.

Response konflik (HTTP 409) terjadi apabila salah satu record sudah dikunci pihak lain:

```json
{
  "success": false,
  "error": "Resource lock conflict",
  "data": {
    "conflicting_key": { "key": "item_product_id", "value": "102" },
    "locked_by": "service-inventory-a1b2",
    "ttl_remaining": 15
  }
}
```

### POST /api/{project}/lock/release

Melepaskan lock yang sebelumnya di-acquire. Hanya pemegang `lock_token` yang dapat melepaskan lock.

| Field | Tipe | Wajib | Keterangan |
|-------|------|-------|------------|
| `resource` | string | Ya | Nama resource (harus sama dengan saat acquire) |
| `lock_keys` | array | Ya | Daftar record yang dilepaskan |
| `lock_token` | string | Ya | Token dari response acquire untuk verifikasi kepemilikan |

Response sukses (HTTP 200):

```json
{
  "success": true,
  "data": { "resource": "item_products", "released_keys": 3, "total_keys": 3 }
}
```

| `released_keys` | Penjelasan |
|-----------------|------------|
| Sama dengan `total_keys` | Semua lock berhasil dilepaskan |
| Kurang dari `total_keys` | Sebagian lock sudah expire atau `lock_token` tidak cocok |
| `0` | `lock_token` salah, atau semua lock sudah expire |

### POST /api/{project}/lock/status

Mengecek status lock pada satu atau lebih record. Berguna untuk debugging atau memeriksa ketersediaan resource sebelum acquire.

| Field | Tipe | Wajib | Keterangan |
|-------|------|-------|------------|
| `resource` | string | Ya | Nama resource |
| `lock_keys` | array | Ya | Daftar record yang dicek |

Response (HTTP 200):

```json
{
  "success": true,
  "data": {
    "resource": "item_products",
    "keys": [
      { "key": "item_product_id", "value": "101", "locked": true, "owner": "service-checkout-d4e5", "ttl_remaining": 25 },
      { "key": "item_product_id", "value": "104", "locked": false, "owner": null, "ttl_remaining": null }
    ]
  }
}
```

---

## Hal yang Harus Diperhatikan (Important Considerations)

| Aspek | Keterangan |
|-------|------------|
| Release di `finally` | Tempatkan release di `finally` block agar lock tidak tertahan ketika operasi gagal |
| Simpan `lock_token` | Tanpa token yang benar, lock tidak dapat dilepaskan manual. Lock tetap expire otomatis setelah TTL habis |
| Sesuaikan TTL | TTL terlalu pendek menyebabkan lock expire sebelum operasi selesai. TTL terlalu panjang menahan record lebih lama saat proses crash |
| All-or-nothing | Apabila sebagian record berkonflik, lock yang sudah didapat otomatis di-rollback |
| Cooperative locking | Lock bersifat kooperatif. Operasi CRUD standar tidak otomatis memeriksa resource lock, sehingga semua pihak harus konsisten menggunakan API ini |
| Maksimum 100 keys | Satu request acquire mengunci maksimum 100 record. Untuk lebih dari itu, pecah menjadi beberapa batch (tidak atomic antar batch) |

---

## Pola Penggunaan dengan Composite Endpoints

Per-record lock otomatis **tidak melindungi** `createComposite`, karena header record belum exist saat INSERT sehingga tidak ada PK yang dapat di-lock. Untuk skenario `createComposite` yang melibatkan shared resource (misalnya validasi stok), gunakan Resource Lock API dari sisi client:

```
createComposite:
  ┌─ Resource Lock (client-side) ── Lock product IDs
  │   └─ Validasi stok/kuota (eksklusif)
  │   └─ createComposite (server-side) ── INSERT header + detail
  └─ Release Resource Lock
```

Operasi `updateComposite` dan `delete` cascade mendapatkan per-record lock otomatis pada header PK. Resource lock diperlukan hanya apabila operasi tersebut juga melibatkan validasi cross-resource.
