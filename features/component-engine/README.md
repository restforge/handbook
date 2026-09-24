# Component Engine

Component Engine berfungsi untuk menjalankan custom logic secara otomatis sebelum dan sesudah operasi CRUD, melalui handler yang didaftarkan di payload JSON.

Semua event bersifat blocking dan berperilaku seperti database trigger: jika handler gagal, seluruh transaction di-rollback. Konfigurasi dilakukan sepenuhnya secara deklaratif, tanpa modifikasi code runtime.

---

## Mendaftarkan Handler di Payload

Tambahkan property `components` di payload JSON untuk mendaftarkan handler pada sebuah tabel.

```json
{
  "tableName": "supplier",
  "primaryKey": "supplier_id",
  "fieldName": ["supplier_id", "supplier_code", "supplier_name"],
  "action": { "create": true, "update": true, "delete": true },
  "components": [
    {
      "properties": {
        "filename": "components/handlers/supplier-handler.js",
        "methods": [
          {
            "name": "validateBeforeInsert",
            "events": "onBeforeInsert",
            "params": [
              { "value": "{tableName}" },
              { "value": "{requestData}" },
              { "value": "{oldData}" },
              { "value": "{newData}" }
            ]
          },
          {
            "name": "syncAfterUpdate",
            "events": "onAfterUpdate",
            "params": [
              { "value": "{tableName}" },
              { "value": "{requestData}" },
              { "value": "{oldData}" },
              { "value": "{newData}" }
            ]
          }
        ]
      }
    }
  ]
}
```

Setelah payload diperbarui, lakukan regenerate module dan restart runtime.

> **Perlu diketahui:** Jika payload tidak memiliki property `components`, semua operasi CRUD berjalan normal tanpa event. Tidak ada perubahan behavior pada modul yang sudah ada.

### Event yang Tersedia

| Event | Dipanggil |
|-------|-----------|
| `onBeforeInsert` | Sebelum INSERT — `oldData` null, `newData` null |
| `onAfterInsert` | Setelah INSERT — `oldData` null, `newData` berisi record baru |
| `onBeforeUpdate` | Sebelum UPDATE — `oldData` berisi data lama, `newData` null |
| `onAfterUpdate` | Setelah UPDATE — `oldData` berisi data lama, `newData` berisi data baru |
| `onBeforeDelete` | Sebelum DELETE — `oldData` berisi data lama, `newData` null |
| `onAfterDelete` | Setelah DELETE — `oldData` berisi data lama, `newData` null |

### Template Variable

Parameter `params` menggunakan template variable yang di-resolve dari context request.

| Variable | Isi |
|----------|-----|
| `{tableName}` | Nama tabel yang sedang dioperasikan |
| `{requestData}` | Data dari request body (`req.body`) |
| `{oldData}` | Data sebelum operasi. `null` untuk INSERT |
| `{newData}` | Data setelah operasi. `null` untuk onBefore dan DELETE |
| `{operation}` | Tipe operasi: `INSERT`, `UPDATE`, atau `DELETE` |
| `{user_id}` | User ID dari request header |
| `{timestamp}` | ISO timestamp saat event dieksekusi |
| `{record_id}` | Primary key record yang dioperasikan |

---

## Menulis File Handler

Letakkan file handler di `src/components/handlers/` pada folder project. Component engine akan melakukan `require()` secara otomatis saat server startup, tanpa registrasi tambahan.

```
project/
  src/
    components/
      handlers/
        supplier-handler.js   ← File handler
  payload/
    supplier.json             ← Payload dengan konfigurasi components
```

Setiap function handler menerima lima parameter, dengan `services` di-inject secara otomatis sebagai parameter terakhir.

```js
async function validateBeforeInsert(table_name, request_data, old_data, new_data, services) {
  const { db, logger } = services;

  if (!request_data.supplier_code) {
    return { success: false, message: 'Supplier code wajib diisi' };
  }

  logger.info({ table: table_name }, 'Validation passed');
  return { success: true };
}

module.exports = { validateBeforeInsert };
```

### Return Value

Handler wajib mengembalikan object dengan property `success`.

```js
// Lanjutkan operasi
return { success: true };

// Rollback transaction dan kembalikan error ke client
return { success: false, message: 'Supplier masih memiliki purchase order aktif' };
```

> **Perlu diketahui:** `return { success: false }` berlaku sama untuk onBefore maupun onAfter — keduanya menyebabkan rollback seluruh transaction. Exception yang tidak tertangkap di dalam handler juga memicu rollback.

### Mendaftarkan Beberapa Handler pada Satu Event

Untuk mendaftarkan beberapa handler pada satu event, tambahkan item di array `methods` dalam urutan eksekusi yang diinginkan.

```json
"methods": [
  { "name": "validateStock", "events": "onBeforeInsert", "params": [...] },
  { "name": "auditLog",      "events": "onBeforeInsert", "params": [...] }
]
```

Handler dieksekusi secara berurutan. Jika handler pertama mengembalikan `{ success: false }`, handler berikutnya tidak dieksekusi.

---

## Mengakses Services di Handler

Component engine meng-inject object `services` sebagai parameter terakhir secara otomatis, tanpa perubahan konfigurasi payload.

```js
async function myHandler(table_name, request_data, old_data, new_data, services) {
  const { db, logger, redis, kafka, cache } = services;

  // Query ke tabel lain
  const result = await db.executeQuery(
    'SELECT COUNT(*) AS total FROM purchase_order WHERE supplier_id = $1',
    [old_data.supplier_id]
  );

  logger.info({ total: result.rows[0].total }, 'Dependency check');
  return { success: true };
}
```

| Service | Key | Tersedia |
|---------|-----|----------|
| Database | `db` | Selalu. Auto-route berdasarkan `DB_TYPE` (PostgreSQL, MySQL, Oracle, SQLite) |
| Logger | `logger` | Selalu. Fallback ke `console` jika pino tidak tersedia |
| Redis | `redis` | `null` jika Redis tidak dikonfigurasi |
| Kafka | `kafka` | `null` jika `KAFKA_ENABLED` bukan `'true'` |
| Cache | `cache` | `null` jika cache tidak dikonfigurasi |

> **Perlu diketahui:** Service opsional (`redis`, `kafka`, `cache`) bernilai `null` jika tidak dikonfigurasi. Lakukan null-check sebelum menggunakannya. Handler lama yang tidak mendeklarasikan parameter `services` tetap berfungsi normal karena JavaScript mengabaikan argumen ekstra.

---

## Memakai Event Composite

Event composite tersedia untuk operasi `/create-composite` dan `/update-composite`. Event ini menggunakan nama berbeda dari event single table untuk menghindari ambiguitas context.

| Event | Dipanggil |
|-------|-----------|
| `onBeforeCompositeInsert` | Sebelum composite INSERT — menerima header + detail items |
| `onAfterCompositeInsert` | Setelah composite INSERT — menerima header + inserted items |
| `onBeforeCompositeUpdate` | Sebelum composite UPDATE — menerima header + old data + detail operations |
| `onAfterCompositeUpdate` | Setelah composite UPDATE — menerima header + old data + detail results |

### Konfigurasi Payload

```json
{
  "components": [
    {
      "properties": {
        "filename": "components/handlers/po-handler.js",
        "methods": [
          {
            "name": "auditAfterCreate",
            "events": "onAfterCompositeInsert",
            "params": [
              { "value": "{tableName}" },
              { "value": "{requestData}" },
              { "value": "{newData}" },
              { "value": "{insertedItems}" },
              { "value": "{detailTable}" }
            ]
          },
          {
            "name": "validateBeforeUpdate",
            "events": "onBeforeCompositeUpdate",
            "params": [
              { "value": "{tableName}" },
              { "value": "{oldData}" },
              { "value": "{detailOperations}" }
            ]
          }
        ]
      }
    }
  ]
}
```

### Contoh Handler

```js
// components/handlers/po-handler.js

async function auditAfterCreate(table_name, request_data, new_data, inserted_items, detail_table) {
  console.log(`[AUDIT] ${inserted_items.length} items inserted in ${detail_table}`);
  return { success: true };
}

async function validateBeforeUpdate(table_name, old_data, detail_operations) {
  if (old_data && old_data.status !== 'draft') {
    return {
      success: false,
      message: `Record tidak dapat diubah karena status sudah '${old_data.status}'`
    };
  }
  return { success: true };
}

module.exports = { auditAfterCreate, validateBeforeUpdate };
```

### Context Object Event Composite

**onBeforeCompositeInsert** — field yang tersedia via template variable:

| Field | Isi |
|-------|-----|
| `{requestData}` | Header data dari request (tanpa detail array) |
| `{oldData}` | `null` — INSERT tidak memiliki data lama |
| `{newData}` | `null` — belum di-insert |
| `{detailItems}` | Array detail items yang akan di-insert |
| `{detailTable}` | Nama tabel detail |
| `{foreignKey}` | Foreign key header-detail |
| `{operation}` | `'COMPOSITE_INSERT'` |

**onAfterCompositeInsert** — field yang tersedia via template variable:

| Field | Isi |
|-------|-----|
| `{requestData}` | Header data asli dari request |
| `{oldData}` | `null` — INSERT tidak memiliki data lama |
| `{newData}` | Header yang sudah di-insert |
| `{insertedItems}` | Array detail items yang sudah di-insert |
| `{detailTable}` | Nama tabel detail |
| `{record_id}` | Primary key header yang baru di-insert |

**onBeforeCompositeUpdate** — field yang tersedia via template variable:

| Field | Isi |
|-------|-----|
| `{requestData}` | Header data dari request |
| `{oldData}` | Header sebelum update |
| `{newData}` | `null` — belum di-update |
| `{detailOperations}` | `{ insert: [], update: [], delete: [] }` |

**onAfterCompositeUpdate** — field yang tersedia via template variable:

| Field | Isi |
|-------|-----|
| `{requestData}` | Header data dari request |
| `{oldData}` | Header sebelum update |
| `{newData}` | Header setelah update |
| `{detailResults}` | `{ inserted: [], updated: [], deleted: [] }` |
| `{record_id}` | Primary key header |

> **Perlu diketahui:** Hook composite hanya berlaku di level header. Tidak ada hook per-item detail (`onBeforeDetailInsert` tidak tersedia). Event `onBeforeCompositeDelete` juga tidak tersedia karena endpoint `/delete-composite` belum ada.

---

## Menangani Error Umum

| Problem | Penyebab umum | Penanganan |
|---------|---------------|------------|
| Handler tidak tereksekusi | Nama event salah di `events`, atau nama function tidak cocok dengan `module.exports` | Verifikasi ejaan event dan nama function di payload vs file handler |
| Handler tidak ditemukan | Path `filename` tidak sesuai lokasi file | Path bersifat relatif terhadap folder `src/` pada project |
| onAfter gagal dan operasi utama ikut di-rollback | Behavior yang diharapkan — semua event bersifat blocking | Pastikan logic handler stabil; untuk side effect fire-and-forget, gunakan Kafka |
| `Unknown operation: create` | Mapping operation name tidak sesuai versi runtime | Pastikan menggunakan versi terbaru restforge-cli |

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| Konfigurasi | Property `components` di payload JSON |
| Lokasi file handler | `src/components/handlers/` |
| Signature handler | `async function(table_name, request_data, old_data, new_data, services)` |
| Services tersedia | `{ db, logger, redis, kafka, cache }` — di-inject otomatis |
| Rollback | `return { success: false }` atau throw exception — berlaku untuk semua event |
| Eksekusi beberapa handler | Sequential sesuai urutan di array `methods` |
| Database | PostgreSQL, MySQL, Oracle, SQLite — output identik (lowercase keys) |
| Import/export | Tidak dicakup component engine — gunakan `importConfig` dan `exportQuery` |

**Lihat juga**: [`catalogs/rdf/`](../../catalogs/rdf/) · [`features/`](../)
