# Field Processor (`processor`)

Alternatif struktur RDF untuk endpoint custom non-CRUD. RDF jenis ini tidak memiliki `tableName`, `fieldName`, atau `action` standar. Setiap entry di `processor` mendefinisikan satu endpoint, baik dengan SQL custom maupun tanpa SQL (lihat section [Processor Tanpa SQL](#processor-tanpa-sql)).

```json
{
    "description": "Sales Order custom endpoints",
    "processor": [
        {
            "name": "submit-order",
            "method": "POST",
            "description": "Submit draft order menjadi pending approval",
            "sql": {
                "query": "UPDATE sales.sales_order SET status = 'pending_approval' WHERE so_id = $1 AND status = 'draft'",
                "params": ["so_id"]
            },
            "request": {
                "body": {
                    "so_id": { "type": "uuid", "required": true },
                    "notes": { "type": "string", "required": false, "maxLength": 200 }
                },
                "headers": {
                    "X-App-Code": { "type": "string", "required": true, "mapTo": "app_code" }
                }
            },
            "response": {
                "message": {
                    "success": "Sales order berhasil di-submit untuk approval.",
                    "empty":   "Sales order tidak ditemukan atau bukan berstatus draft.",
                    "error":   "Gagal submit sales order."
                }
            }
        }
    ]
}
```

## Properti `processor[]`

| Property | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `name` | string | Ya | Nama endpoint (dipasang sebagai `/{resource}/{name}`) |
| `method` | enum | Ya | HTTP method: `GET`, `POST`, `PUT`, `PATCH`, `DELETE` |
| `description` | string | Tidak | Deskripsi endpoint untuk dokumentasi |
| `sql` | object | Tidak | Blok SQL. Bila absen, processor di-generate dalam mode tanpa SQL (lihat di bawah) |
| `sql.query` | string | Kondisional | SQL statement inline dengan placeholder `$1, $2, ...` (PostgreSQL style). Bila blok `sql` ada, salah satu dari `sql.query` atau `sql.file` harus diisi |
| `sql.file` | string | Kondisional | Path file SQL eksternal, relatif terhadap folder file payload. Alternatif dari `sql.query` |
| `sql.params` | array | Tidak | Daftar nama field yang diikat ke placeholder. Nilai diambil dari gabungan input (body, route params, query string, dan header hasil `mapTo`); bila input kosong, jatuh ke properti `default` pada definisi field |
| `request.body` | object | Tidak | Skema validasi body request (lihat tabel properti field di bawah) |
| `request.params` | object | Tidak | Skema route parameter. Setiap key ditambahkan ke path endpoint sebagai segment `/:key` |
| `request.headers` | object | Tidak | Skema validasi headers (dengan `mapTo` untuk penggantian nama field di input) |
| `request.validate` | boolean | Tidak | Default `true`. Bila `false`, seluruh validasi di router di-opt-out dan validasi menjadi tanggung jawab file processor |
| `response.message` | object | Tidak | Pesan response untuk skenario `success`, `empty`, `error`. Pesan `empty` hanya dipakai mode SQL (hasil query kosong, HTTP 200); scaffold tanpa SQL hanya memakai `success` dan `error` |
| `cache.enabled` | boolean | Tidak | Default `false`. Aktifkan response caching untuk processor ini. Diabaikan dengan pesan WARN bila method mutasi (`PUT`, `DELETE`, `PATCH`) atau SQL mengandung statement mutasi |
| `cache.ttl` | number | Tidak | Masa simpan cache dalam detik. Default `300` |

## Properti Field pada `request`

Berlaku untuk definisi field di `request.body`, `request.params`, dan `request.headers`:

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `type` | enum | `string`, `number`, `integer`, `boolean`, `uuid`, `array`, `object`, `date`, `datetime`. Catatan: `date` dan `datetime` diterima generator namun belum menghasilkan pengecekan type di router |
| `required` | boolean | Field wajib ada di input; bila absen, router menolak dengan HTTP 400 |
| `format` | enum | Validasi format via regex whitelist: `email`, `url`, `phone-id`, `uuid` |
| `enum` | array | Whitelist nilai yang diizinkan |
| `minLength` | integer | Panjang minimum string |
| `maxLength` | integer | Panjang maksimum string |
| `sensitive` | boolean | Bila `true`, nilai field di-mask `***MASKED***` pada debug log router. Field dengan nama umum sensitif (`password`, `token`, `secret`, `api_key`, dll.) sudah ter-mask otomatis tanpa flag ini |
| `default` | any | Nilai fallback saat binding `sql.params` bila input kosong |
| `mapTo` | string | Khusus `request.headers`: nama field pengganti di object input |

Seluruh validasi di atas di-enforce di router layer (kode auto-generated). Kegagalan validasi mengembalikan HTTP 400 dengan array `errors` berisi pasangan `field` dan `message` per pelanggaran.

Detail lengkap dibahas di dokumentasi fitur processor terpisah.

## Processor Tanpa SQL

Blok `sql` bersifat opsional. Payload yang hanya berisi `name`, `method`, dan (opsional) `request` tetap valid dan menghasilkan endpoint lengkap:

- Route tetap terdaftar di router beserta request validation sesuai skema `request`.
- File processor di-generate sebagai scaffold implementasi manual, dengan destructuring input dari skema `request` dan body function yang siap diisi business logic custom.

Mode ini ditujukan untuk workflow "payload sebagai register routing, business logic ditulis manual di file processor". File implementasi yang sudah diedit tidak akan ditimpa pada re-run `processor create` tanpa `--force` (lihat dokumentasi command [`processor create`](../../commands/restforge-backend/processor/create.md)).

```json
{
    "description": "Visitor seeder endpoints",
    "processor": [
        {
            "name": "seed-visitors",
            "method": "POST",
            "description": "Seed data visitor, implementasi manual di file processor",
            "request": {
                "body": {
                    "count": { "type": "integer", "required": true }
                }
            }
        }
    ]
}
```

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
