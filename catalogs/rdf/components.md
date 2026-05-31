# Field Component Engine (`components`)

Konfigurasi event lifecycle hooks pada operasi CRUD. Setiap hook merujuk ke method di file handler eksternal.

```json
{
    "components": [
        {
            "properties": {
                "filename": "components/supplier-hooks.js",
                "methods": [
                    {
                        "name": "validateSupplierCode",
                        "events": "onBeforeInsert",
                        "params": [
                            { "value": "{requestData}" },
                            { "value": "{user_id}" }
                        ]
                    },
                    {
                        "name": "notifySlack",
                        "events": "onAfterInsert"
                    }
                ]
            }
        }
    ]
}
```

## Properti `components[].properties`

| Property | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `filename` | string | Ya | Path file handler relatif dari root project |
| `methods` | array | Ya | Daftar method handler beserta event yang men-trigger |

## Properti `methods[]`

| Property | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `name` | string | Ya | Nama function yang di-export dari file handler |
| `events` | enum | Ya | Event hook (lihat tabel di bawah) |
| `params` | array | Tidak | Daftar template variable yang di-pass ke handler |

## Event Hook yang Didukung

| Event | Trigger |
|-------|---------|
| `onBeforeInsert`, `onAfterInsert` | Endpoint `/create` |
| `onBeforeUpdate`, `onAfterUpdate` | Endpoint `/update` |
| `onBeforeDelete`, `onAfterDelete` | Endpoint `/delete` |
| `onBeforeCompositeInsert`, `onAfterCompositeInsert` | Endpoint `/create-composite` |
| `onBeforeCompositeUpdate`, `onAfterCompositeUpdate` | Endpoint `/update-composite` |

## Template Variable di `params[].value`

| Variable | Isi |
|----------|-----|
| `{tableName}` | Nama tabel resource |
| `{requestData}` | Body request lengkap |
| `{oldData}` | Data sebelum operasi (`update`, `delete`) |
| `{newData}` | Data setelah operasi (`update`, `create`) |
| `{operation}` | Nama operasi (`insert`/`update`/`delete`) |
| `{user_id}` | User ID dari context request |
| `{timestamp}` | Timestamp eksekusi |
| `{record_id}` | Primary key record yang dioperasikan |

Detail lengkap (lifecycle flow, error handling, integrasi dengan distributed lock) dibahas di dokumentasi fitur component engine terpisah.

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
