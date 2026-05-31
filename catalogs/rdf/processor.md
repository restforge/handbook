# Field Processor (`processor`)

Alternatif struktur RDF untuk endpoint custom non-CRUD. RDF jenis ini tidak memiliki `tableName`, `fieldName`, atau `action` standar. Setiap entry di `processor` mendefinisikan satu endpoint dengan SQL custom.

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
| `name` | string | Ya | Nama endpoint (di-mount sebagai `/{resource}/{name}`) |
| `method` | enum | Ya | HTTP method: `GET`, `POST`, `PUT`, `PATCH`, `DELETE` |
| `description` | string | Tidak | Deskripsi endpoint untuk dokumentasi |
| `sql.query` | string | Ya | SQL statement dengan placeholder `$1, $2, ...` (PostgreSQL style) |
| `sql.params` | array | Ya | Daftar nama field di request body yang di-bind ke placeholder |
| `request.body` | object | Tidak | Skema validasi body request |
| `request.headers` | object | Tidak | Skema validasi headers (dengan `mapTo` untuk re-naming) |
| `response.message` | object | Tidak | Pesan response untuk skenario `success`, `empty`, `error` |

Detail lengkap dibahas di dokumentasi fitur processor terpisah.

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
