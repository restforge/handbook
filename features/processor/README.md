# Processor

> Custom endpoint dengan business logic manual, di-scaffold dari payload JSON.

Processor berfungsi untuk membuat endpoint yang tidak bisa diekspresikan dengan CRUD standar — seperti submit order, login, approval flow, atau kalkulasi kompleks. Generator menghasilkan router dan scaffold awal; business logic ditulis secara manual di file processor yang tidak pernah ditimpa.

---

## Perbandingan dengan CRUD Module

| Aspek | CRUD Module | Processor |
|-------|-------------|-----------|
| Tujuan | Operasi CRUD standar | Endpoint dengan business logic custom |
| Logic | Sepenuhnya di-generate | Scaffold di-generate, logic ditulis manual |
| Regenerasi aman | Ya | Ya — file processor tidak di-overwrite |
| Contoh | Data user, role, produk | Login, submit order, approval, kalkulasi |

---

## Quick Start

### 1. Buat payload processor

Definisikan endpoint di file `payload/`.

```json
{
  "processor": [
    {
      "name": "submit-order",
      "method": "POST",
      "sql": {
        "query": "SELECT so_id, status FROM sales.order WHERE so_id = $1 AND status = 'draft'",
        "params": ["so_id"]
      },
      "request": {
        "body": {
          "so_id":  { "type": "uuid",   "required": true  },
          "notes":  { "type": "string", "required": false }
        }
      }
    }
  ]
}
```

### 2. Generate scaffold

```bash
npx restforge processor create --project=sales --name=order-process --payload=order_process.json
```

Dua file dihasilkan:

```
src/modules/sales/
├── order-process.js                            ← Router — JANGAN diubah
└── processor/
    └── order-process/
        └── submit-order.js                     ← Tulis business logic di sini
```

### 3. Tulis business logic

Buka `src/modules/sales/processor/order-process/submit-order.js` dan implementasikan method `process`.

```js
const processor = {
  async process(input, services) {
    const { db } = services;

    const result = await db.executeQuery(
      'SELECT so_id, status, created_by FROM sales.order WHERE so_id = $1',
      [input.so_id]
    );
    const order = result.rows?.[0];

    if (!order) return { success: false, statusCode: 404, message: 'Order tidak ditemukan.' };
    if (order.status !== 'draft') return { success: false, statusCode: 422, message: `Status saat ini: ${order.status}.` };

    await db.executeQuery(
      'UPDATE sales.order SET status = $1 WHERE so_id = $2',
      ['pending_approval', input.so_id]
    );

    return { success: true, message: 'Order berhasil di-submit.' };
  }
};

module.exports = processor;
```

> **Perlu diketahui:** File processor tidak di-overwrite oleh generator. Jalankan ulang perintah generate kapan saja setelah mengubah payload — custom logic tetap aman.

---

## Dokumentasi Lengkap

| Dokumen | Isi |
|---------|-----|
| [Payload Processor](payload.md) | Format payload, properti request schema, SQL dari file eksternal |
| [Menulis Processor](handler.md) | Signature, services, return value, contoh lengkap |
| [Konvensi Penamaan](naming.md) | Aturan penamaan `--project`, `--name`, `processor[].name`, suffix `-process` |
| [Keamanan](security.md) | Parameterized query, validasi business rule, masking log, checklist pre-merge |
