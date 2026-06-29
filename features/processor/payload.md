# Payload Processor

> Format payload JSON untuk mendefinisikan endpoint, request schema, dan SQL awal.

Kembali ke [ikhtisar Processor](README.md).

---

## Struktur Dasar

Setiap item dalam array `processor[]` mendefinisikan satu endpoint beserta route-nya.

```json
{
  "description": "Sales Order processing endpoints",
  "processor": [
    {
      "name": "submit-order",
      "method": "POST",
      "description": "Submit draft order untuk approval",
      "sql": {
        "query": "SELECT so_id, status FROM sales.order WHERE so_id = $1 AND status = 'draft'",
        "params": ["so_id"]
      },
      "request": {
        "body": {
          "so_id":  { "type": "uuid",   "required": true,  "description": "ID Sales Order" },
          "notes":  { "type": "string", "required": false, "description": "Catatan submit"  }
        },
        "params": {
          "id": { "type": "uuid", "required": true }
        },
        "headers": {
          "X-App-Code": { "type": "string", "mapTo": "app_code", "description": "Kode aplikasi" }
        }
      },
      "response": {
        "message": {
          "success": "Order berhasil di-submit.",
          "empty":   "Order tidak ditemukan atau bukan berstatus draft.",
          "error":   "Gagal submit order."
        }
      }
    }
  ]
}
```

---

## Properti Payload

| Properti | Wajib | Keterangan |
|----------|-------|------------|
| `processor[].name` | Ya | Nama endpoint — menjadi nama file processor dan segmen URL terakhir |
| `processor[].method` | Ya | HTTP method: `GET`, `POST`, `PUT`, `DELETE`, `PATCH` |
| `processor[].description` | Tidak | Deskripsi singkat endpoint |
| `processor[].sql.query` | Tidak | Query SQL awal yang di-embed di scaffold |
| `processor[].sql.params` | Tidak | Array nama field untuk binding query di atas |
| `processor[].sql.file` | Tidak | Alternatif: path ke file `.sql` eksternal |
| `processor[].request.body` | Tidak | Definisi field dari request body |
| `processor[].request.params` | Tidak | Definisi URL params — menjadi `/:param` di route |
| `processor[].request.headers` | Tidak | Definisi header dengan `mapTo` untuk mapping ke `input` |
| `processor[].response.message` | Tidak | Pesan scaffold untuk kondisi success, empty, dan error |

---

## Request Schema

Definisi field di `request.body`, `request.params`, dan `request.headers` mengontrol validasi otomatis di router. Request yang tidak memenuhi schema ditolak dengan HTTP 400 sebelum sampai ke processor.

### Properti per Field

| Properti | Nilai | Keterangan |
|----------|-------|------------|
| `type` | `string`, `number`, `integer`, `boolean`, `uuid`, `date`, `datetime`, `array`, `object` | Tipe data yang di-enforce |
| `required` | `true` / `false` | Field absent atau kosong di-reject jika `true` |
| `format` | `email`, `url`, `phone-id`, `uuid` | Regex check tambahan pada field bertipe `string` |
| `minLength` | number | Panjang minimum untuk field bertipe `string` |
| `maxLength` | number | Panjang maksimum untuk field bertipe `string` |
| `enum` | array | Whitelist nilai yang diizinkan |
| `sensitive` | `true` / `false` | Paksa masking field di debug log, meski nama tidak cocok pattern default |
| `description` | string | Keterangan field (masuk ke scaffold komentar) |

### Contoh Schema Lengkap

```json
"request": {
  "body": {
    "so_id":       { "type": "uuid",   "required": true  },
    "status":      { "type": "string", "required": true,  "enum": ["draft", "pending"] },
    "email":       { "type": "string", "required": false, "format": "email" },
    "notes":       { "type": "string", "required": false, "maxLength": 500 },
    "ktp_number":  { "type": "string", "required": false, "sensitive": true }
  }
}
```

> **Perlu diketahui:** Validasi router menangani aspek mekanis (tipe, required, format, enum, panjang). Validasi business rule — status transition, authorization, konsistensi antar entitas — tetap tanggung jawab processor. Lihat [Keamanan](security.md) untuk detail lebih lanjut.

### Format Error Validasi

```json
{
  "success": false,
  "statusCode": 400,
  "message": "Validation failed",
  "errors": [
    { "field": "so_id",    "message": "Field \"so_id\" wajib diisi" },
    { "field": "app_code", "message": "Header X-App-Code wajib diisi" }
  ],
  "timestamp": "2026-04-22T10:00:00.000Z"
}
```

### Header Mapping

Field `mapTo` memetakan nilai header ke nama field di `input` processor.

```json
"headers": {
  "X-App-Code": {
    "type": "string",
    "required": true,
    "mapTo": "app_code",
    "enum": ["sales-app", "inventory-app"]
  }
}
```

Di processor, nilai header tersedia sebagai `input.app_code`.

### Opt-Out Validasi Router

Use case seperti dynamic search yang tidak bisa dienumerasi di schema dapat menonaktifkan validasi router via `validate: false`.

```json
"request": {
  "validate": false,
  "body": {
    "search_by":    { "type": "string", "required": true },
    "search_value": { "type": "string", "required": true }
  }
}
```

> **Perlu diketahui:** Saat `validate: false`, seluruh validasi mekanis menjadi tanggung jawab processor. Sensitive field masking di debug log tetap berfungsi. Lihat [Keamanan — Opt-Out Validasi](security.md#opt-out-validasi-router).

---

## SQL dari File Eksternal

Query panjang dapat disimpan di file `.sql` terpisah untuk keterbacaan.

```json
"sql": {
  "file": "query/submit-order.sql",
  "params": ["so_id", "app_code"]
}
```

File SQL disimpan di `payload/query/` relatif terhadap payload. Scaffold processor akan me-load file tersebut via `fs.readFileSync` pada saat dieksekusi.

---

## Perilaku Generator

| File | Kondisi | Perilaku |
|------|---------|----------|
| Router `{name}.js` | Ada maupun tidak | Selalu di-overwrite |
| Processor `processor/{name}/{endpoint}.js` | Belum ada | Dibuat baru |
| Processor `processor/{name}/{endpoint}.js` | Sudah ada | Di-skip — tidak di-overwrite |
| Processor `processor/{name}/{endpoint}.js` | Sudah ada + `--force` | Di-overwrite (file lama diarsip) |

> **Perlu diketahui:** Router selalu di-overwrite agar schema validasi dan sensitive masking selalu sinkron dengan payload. Jangan pernah menulis custom logic di file router.

---

Lihat juga: [Menulis Processor](handler.md) · [Konvensi Penamaan](naming.md) · [Keamanan](security.md)
