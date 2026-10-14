# Field Aggregate (`aggregateConfig`)

Konfigurasi JOIN untuk endpoint `/aggregate`. JOIN didefinisikan di RDF (bukan di request body) untuk alasan keamanan.

```json
{
    "action": { "aggregate": true },
    "aggregateConfig": {
        "joins": {
            "warehouse": {
                "tableName": "warehouse",
                "joinType": "LEFT",
                "sourceField": "warehouse_id",
                "targetField": "warehouse_id",
                "fields": ["warehouse_code", "warehouse_name"]
            }
        }
    }
}
```

## Properti `joins[name]`

| Property | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| *(key)* | string | Ya | Nama join yang direferensikan dari request body (`joins: ["warehouse"]`) |
| `tableName` | string | Ya | Nama tabel yang di-join |
| `joinType` | enum | Ya | `INNER`, `LEFT`, `RIGHT`, atau `FULL` |
| `sourceField` | string | Ya | Kolom di tabel utama (sisi kiri join) |
| `targetField` | string | Ya | Kolom di tabel join (sisi kanan join) |
| `fields` | array | Ya | Kolom dari tabel join yang boleh diakses hanya di `group_by` (dan `select`) request. Tidak boleh dipakai di `operations` maupun `having` |

Detail lengkap (operasi `count`/`sum`/`avg`/`min`/`max`, `having` clause, format response) dibahas di dokumentasi fitur aggregate terpisah.

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
