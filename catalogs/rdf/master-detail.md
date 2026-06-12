# Field Master Detail (`masterDetail`)

Konfigurasi untuk operasi composite (header + detail dalam satu transaksi). Wajib ditetapkan jika action `createComposite`, `updateComposite`, atau `readComposite` aktif.

```json
{
    "action": {
        "createComposite": true,
        "updateComposite": true,
        "readComposite": true
    },
    "masterDetail": {
        "enabled": true,
        "detailTable": "stock_inbound_item",
        "foreignKey": "stock_inbound_id",
        "cascadeDelete": true,
        "transactionMode": "required",
        "detailConfig": {
            "tableName": "stock_inbound_item",
            "primaryKey": "stock_inbound_item_id",
            "fieldName": [
                "stock_inbound_item_id", "stock_inbound_id",
                "line_number", "item_product_id", "qty_received",
                "unit_price", "total_amount"
            ],
            "detailQuery": "file:query/stock-inbound-detail.sql",
            "requiredFields": ["line_number", "item_product_id", "qty_received", "unit_price"],
            "autoCalculateFields": {
                "total_amount": {
                    "type": "generated",
                    "formula": "qty_received * unit_price"
                }
            }
        },
        "headerCalculations": {
            "total_items":  { "type": "count", "source": "items.length" },
            "total_qty":    { "type": "sum",   "source": "items.qty_received" },
            "total_amount": { "type": "sum",   "source": "items.total_amount" }
        }
    }
}
```

## Properti Top-Level `masterDetail`

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `enabled` | boolean | Mengaktifkan fitur master-detail |
| `detailTable` | string | Nama tabel detail di database |
| `foreignKey` | string | Kolom FK di tabel detail yang merujuk ke header |
| `cascadeDelete` | boolean | Jika `true`, hapus header menghapus seluruh detail items |
| `transactionMode` | string | Hanya nilai `"required"` yang didukung. Semua operasi dalam satu transaction |
| `detailConfig` | object | Konfigurasi tabel detail (lihat di bawah) |
| `headerCalculations` | object | Field header yang dihitung secara otomatis dari detail items |

## Properti `detailConfig`

| Property | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `tableName` | string | Ya | Nama tabel detail (umumnya sama dengan `masterDetail.detailTable`) |
| `primaryKey` | string | Ya | PK tabel detail. Wajib bertipe VARCHAR UUID-compatible |
| `fieldName` | array | Ya | Daftar kolom detail yang dikelola |
| `detailQuery` | string | Ya | SQL SELECT untuk membaca detail. Mendukung inline atau `file:` |
| `requiredFields` | array | Tidak | Kolom yang wajib diisi saat insert detail item |
| `autoCalculateFields` | object | Tidak | Field detail yang dihitung otomatis per baris |

## Properti `autoCalculateFields[name]`

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `type` | enum | `"calculated"` (dihitung app code, masuk SQL) atau `"generated"` (database GENERATED COLUMN, dikecualikan dari SQL) |
| `formula` | string | Ekspresi matematika dengan referensi nama kolom (contoh: `qty_received * unit_price`) |
| `description` | string | Deskripsi opsional |

## Properti `headerCalculations[name]`

Field header yang dihitung **oleh application code** dari detail items saat create/update composite.

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `type` | enum | `"count"` atau `"sum"` |
| `source` | string | Referensi sumber. `items.length` untuk count seluruh items, `items.<field>` untuk sum kolom detail tertentu |
| `description` | string | Deskripsi opsional |

Detail lengkap (perbedaan `autoCalculateFields` calculated vs generated, concurrency contract, interaksi dengan `fieldPolicy`) dibahas di dokumentasi fitur CRUD composite terpisah.

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
