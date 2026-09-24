# Menulis Processor

> Signature, services, return value, dan contoh lengkap untuk file processor.

Kembali ke [ikhtisar Processor](README.md).

---

## Lokasi File

Generator menghasilkan dua file dengan peran yang berbeda.

```
src/modules/{project}/
├── {name}.js                                ← Router — JANGAN diubah
└── processor/
    └── {name}/
        └── {endpoint}.js                    ← Tulis business logic di sini
```

File router selalu ditimpa setiap kali generator dijalankan. Semua custom logic yang ditulis di sana akan hilang. File processor tidak ditimpa, sehingga aman untuk dikustomisasi.

---

## Signature

Setiap file processor mengekspor object dengan method `process`.

```js
const processor = {
  async process(input, services, req) {
    // Business logic
    return { success: true, message: '...' };
  }
};

module.exports = processor;
```

### Parameter

| Parameter | Tipe | Keterangan |
|-----------|------|------------|
| `input` | `Object` | Gabungan `req.body`, `req.params`, `req.query`, dan header yang di-`mapTo`. Validasi mekanis sudah lolos saat sampai di sini |
| `services` | `Object` | Services yang di-inject: `{ db, logger, redis, kafka, cache }` |
| `req` | `Object` | Express request object — untuk akses `headers`, `ip`, `method`, `originalUrl` |

Parameter `req` bersifat opsional. Processor yang tidak memerlukan akses HTTP langsung cukup mendeklarasikan `process(input, services)`.

---

## Services

Services yang tersedia melalui destructuring dari parameter kedua.

```js
async process(input, services) {
  const { db, logger, redis, kafka, cache } = services;

  // Query database
  const result = await db.executeQuery('SELECT * FROM tabel WHERE id = $1', [input.id]);

  // Structured logging
  logger.info({ event: 'submit-order', orderId: input.so_id }, 'Order submitted');

  // Redis (jika dikonfigurasi)
  await redis.set('key', 'value');

  // Kafka (jika dikonfigurasi)
  await kafka.send('topic', { event: 'order_submitted' });
}
```

| Service | Key | Tersedia |
|---------|-----|----------|
| Database | `db` | Selalu. Auto-route berdasarkan `DB_TYPE` |
| Logger | `logger` | Selalu. Instance pino |
| Redis | `redis` | `null` jika Redis tidak dikonfigurasi |
| Kafka | `kafka` | `null` jika `KAFKA_ENABLED` bukan `'true'` |
| Cache | `cache` | `null` jika cache tidak dikonfigurasi |

> **Perlu diketahui:** Service opsional (`redis`, `kafka`, `cache`) bernilai `null` jika tidak dikonfigurasi. Lakukan null-check sebelum menggunakannya.

---

## Return Value

Processor mengembalikan object response langsung ke client.

| Properti | Wajib | Default | Keterangan |
|----------|-------|---------|------------|
| `success` | Ya | — | `true` atau `false` |
| `statusCode` | Tidak | `200` | HTTP status code |
| `message` | Ya | — | Pesan response ke client |
| `data` | Tidak | — | Payload data (object atau array) |
| `timestamp` | Tidak | — | ISO 8601 timestamp |

```js
// Sukses dengan data
return {
  success: true,
  statusCode: 200,
  message: 'Order berhasil di-submit.',
  data: { so_id: input.so_id, status: 'pending_approval' },
  timestamp: new Date().toISOString()
};

// Gagal — business rule
return { success: false, statusCode: 422, message: `Status saat ini: ${order.status}.` };

// Gagal — not found
return { success: false, statusCode: 404, message: 'Order tidak ditemukan.' };
```

---

## Mengakses HTTP Request

Gunakan parameter `req` untuk mengakses informasi HTTP yang tidak ada di `input`.

```js
async process(input, services, req) {
  // IP address client (proxy-aware)
  const ip = req.headers['x-forwarded-for']?.split(',')[0]?.trim()
    || req.headers['x-real-ip']
    || req.connection?.remoteAddress;

  // User agent
  const userAgent = req.headers['user-agent'];

  // Authorization header
  const authHeader = req.headers['authorization'];

  // HTTP method dan URL
  const method = req.method;       // 'POST'
  const url    = req.originalUrl;  // '/api/sales/order-process/submit-order'
}
```

---

## Contoh: Query Sederhana

Processor untuk mengambil data produk berdasarkan kode.

```js
const processor = {
  async process(input, services) {
    const { db } = services;

    const result = await db.executeQuery(
      'SELECT product_id, product_code, product_name, stock FROM inventory.product WHERE product_code = $1 AND is_active = true',
      [input.product_code]
    );

    if (!result || result.rows.length === 0) {
      return { success: false, statusCode: 404, message: 'Produk tidak ditemukan.' };
    }

    return {
      success: true,
      message: 'Data produk ditemukan.',
      data: result.rows[0],
      timestamp: new Date().toISOString()
    };
  }
};

module.exports = processor;
```

---

## Contoh: Business Logic dengan Audit Log

Processor submit order dengan validasi status, authorization, dan pencatatan audit.

```js
const processor = {
  async process(input, services, req) {
    const { db, logger } = services;

    // Query order
    const orderResult = await db.executeQuery(
      'SELECT so_id, so_number, status, created_by FROM sales.order WHERE so_id = $1',
      [input.so_id]
    );
    const order = orderResult.rows?.[0];

    if (!order) return { success: false, statusCode: 404, message: 'Order tidak ditemukan.' };

    // Validasi status transition
    if (order.status !== 'draft') {
      return { success: false, statusCode: 422, message: `Order tidak dapat di-submit. Status saat ini: ${order.status}.` };
    }

    // Validasi authorization
    if (order.created_by !== input.submitted_by) {
      return { success: false, statusCode: 403, message: 'Akses ditolak.' };
    }

    // Update status
    await db.executeQuery(
      'UPDATE sales.order SET status = $1, submitted_at = NOW(), submitted_by = $2 WHERE so_id = $3',
      ['pending_approval', input.submitted_by, input.so_id]
    );

    // Audit log
    const ip = req.headers['x-forwarded-for']?.split(',')[0]?.trim() || req.ip;
    await db.executeQuery(
      'INSERT INTO sales.audit_log (so_id, action, performed_by, ip_address, created_at) VALUES ($1, $2, $3, $4, NOW())',
      [input.so_id, 'SUBMIT', input.submitted_by, ip]
    );

    logger.info({ event: 'submit-order', soId: input.so_id }, 'Order submitted');

    return {
      success: true,
      message: 'Order berhasil di-submit untuk approval.',
      data: { so_id: order.so_id, so_number: order.so_number, status: 'pending_approval' },
      timestamp: new Date().toISOString()
    };
  }
};

module.exports = processor;
```

---

## Multi-Processor: Satu Payload, Beberapa Endpoint

Satu file payload dapat mendefinisikan beberapa endpoint sekaligus melalui array `processor[]`. Generator menghasilkan satu router dengan beberapa route dan satu file processor per aksi.

```json
{
  "processor": [
    { "name": "submit-order",  "method": "POST", ... },
    { "name": "approve-order", "method": "POST", ... },
    { "name": "reject-order",  "method": "POST", ... },
    { "name": "cancel-order",  "method": "POST", ... }
  ]
}
```

Hasil generate untuk payload di atas:

```
src/modules/sales/
├── order-process.js                    ← Router (1 file, 4 route)
└── processor/
    └── order-process/
        ├── submit-order.js             ← Business logic per aksi
        ├── approve-order.js
        ├── reject-order.js
        └── cancel-order.js
```

Route yang dihasilkan:

| Method | URL | File Processor |
|--------|-----|---------------|
| `POST` | `/api/sales/order-process/submit-order` | `submit-order.js` |
| `POST` | `/api/sales/order-process/approve-order` | `approve-order.js` |
| `POST` | `/api/sales/order-process/reject-order` | `reject-order.js` |
| `POST` | `/api/sales/order-process/cancel-order` | `cancel-order.js` |

Multi-processor tepat digunakan ketika beberapa aksi secara logis beroperasi pada entitas yang sama dan masing-masing memiliki business logic tersendiri yang tidak bisa di-handle CRUD standar.

---

Lihat juga: [Payload Processor](payload.md) · [Konvensi Penamaan](naming.md) · [Keamanan](security.md)
