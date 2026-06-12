# Field Import Config (`importConfig`)

Konfigurasi endpoint `/import-upload`, `/import-preview`, `/import-commit` untuk import data dari Excel.

```json
{
    "action": { "import": true },
    "importConfig": {
        "enabled": true,
        "upsertKeys": ["supplier_code"],
        "upsertStrategy": "update_existing",
        "requiredFields": ["supplier_code", "supplier_name"],
        "maxFileSize": "10MB",
        "allowedFormats": ["xlsx"],
        "chunkSize": 100,
        "lookupFields": {
            "city_id": {
                "targetField": "city_id",
                "lookupTable": "city",
                "lookupColumn": "city_name",
                "lookupIdColumn": "city_id",
                "required": true
            }
        }
    }
}
```

## Properti Top-Level

| Property | Tipe | Default | Keterangan |
|----------|------|---------|-----------|
| `enabled` | boolean | - | Aktifkan fitur import |
| `upsertKeys` | array | - | Kolom untuk deteksi duplikat |
| `upsertStrategy` | enum | - | `update_existing`, `insert_only`, atau `skip_existing` |
| `requiredFields` | array | - | Kolom wajib diisi saat import |
| `maxFileSize` | string | - | Batas ukuran file upload (contoh: `"10MB"`) |
| `allowedFormats` | array | `["xlsx"]` | Format file yang diizinkan |
| `chunkSize` | integer | `100` | Jumlah baris per batch INSERT/UPDATE |
| `lookupFields` | object | - | Lookup FK saat import (nama kolom Excel → ID database) |

## Properti `lookupFields[name]`

| Property | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `targetField` | string | Ya | Nama kolom di tabel resource |
| `lookupTable` | string | Ya | Tabel sumber lookup |
| `lookupColumn` | string | Ya | Kolom display di tabel sumber (yang muncul di Excel) |
| `lookupIdColumn` | string | Ya | Kolom ID di tabel sumber (yang disimpan ke resource) |
| `required` | boolean | Tidak | Jika `true`, baris ditolak ketika lookup tidak cocok |

Detail lengkap dibahas di dokumentasi fitur export-import terpisah.

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
