# Field Sumber Data (Data Source Fields)

## Prioritas Resolusi

Endpoint `/lookup`, `/read`, dan `/first` menentukan sumber data dengan prioritas tiga tingkat:

```
viewName  →  viewQuery  →  tableName
```

| Prioritas | Field RDF | Digunakan oleh |
|:---------:|-----------|----------------|
| 1 | `viewName` | Lookup, Read, First (SELECT FROM viewName) |
| 2 | `viewQuery` | Lookup, Read, First (dibungkus sebagai subquery) |
| 3 | `tableName` | Fallback terakhir |

Endpoint `/datatables` selalu memakai `datatablesQuery`, terpisah dari resolusi di atas. Endpoint `/export` memakai `exportQuery`, fallback ke `datatablesQuery`.

## `datatablesQuery`

SQL SELECT untuk endpoint `/datatables`. Mendukung dua format:

```json
{
    "datatablesQuery": "SELECT supplier_id, supplier_code, supplier_name FROM supplier"
}
```

```json
{
    "datatablesQuery": "file:query/supplier-datatables.sql"
}
```

| Format | Pola | Keterangan |
|--------|------|-----------|
| Inline SQL | String dimulai dengan `SELECT` | Diparsing apa adanya |
| File reference | String dengan prefix `file:` | Path relatif terhadap folder `payload/` |

## `datatablesWhere`

Array kolom yang dapat dicari di endpoint `/datatables`.

```json
{
    "datatablesWhere": ["supplier_code", "supplier_name", "all"]
}
```

| Item | Perilaku |
|------|----------|
| Nama kolom | Search hanya pada kolom tersebut (jika `search_by` di request match) |
| `"all"` | Mengaktifkan pencarian lintas-kolom (search semua kolom yang terdaftar) |

## `viewQuery`

SQL SELECT alternatif untuk action read-only. Berguna saat resource butuh JOIN ke tabel lain untuk display kolom join (misal `category_name` di endpoint `item_product`).

```json
{
    "viewQuery": "file:query/item-product-detail.sql"
}
```

Behavior generator: `viewQuery` dibungkus sebagai subquery (`SELECT ... FROM (<viewQuery>) a`) sehingga kolom hasil JOIN bisa dipakai di WHERE clause action read-only.

## `viewName`

Nama database VIEW yang sudah ada. Prioritas lebih tinggi dari `viewQuery`.

```json
{
    "viewName": "v_supplier_detail"
}
```

## `exportQuery`

SQL khusus untuk endpoint `/export`. Jika tidak di-set, generator fallback ke `datatablesQuery`.

```json
{
    "exportQuery": "SELECT supplier_code, supplier_name, email, phone, city FROM supplier"
}
```

## `dateTimeFields`

Konfigurasi field bertipe datetime untuk parsing format input client. Dapat ditulis sebagai array (sintaks ringkas) atau object (sintaks penuh dengan format spesifik).

**Sintaks array (default format `dd/MM/yyyy`):**

```json
{
    "dateTimeFields": ["start_date", "end_date"]
}
```

**Sintaks object (format eksplisit per field):**

```json
{
    "dateTimeFields": {
        "visit_date": {
            "type": "timestamp",
            "format": "dd/MM/yyyy HH:mm"
        }
    }
}
```

| Format | Penggunaan |
|--------|-----------|
| `dd/MM/yyyy` | Tanggal tanpa waktu |
| `dd/MM/yyyy HH:mm:ss` | Tanggal + waktu lengkap |
| `dd/MM/yyyy HH:mm` | Tanggal + jam-menit |
| `HH:mm:ss` | Waktu saja |

Detail per dialect dan interaksi dengan `fieldValidation.constraints.format` dibahas di dokumentasi fitur create terpisah, section "Date Parsing".

## `columnFormats`

Format tampilan kolom untuk endpoint `/export` (Excel). Per kolom dapat di-set tipe dan pattern.

```json
{
    "columnFormats": {
        "purchase_price": { "type": "number", "format": "#,##0.00" },
        "register_date":  { "type": "date",   "format": "dd/MM/yyyy" },
        "is_active":      { "type": "boolean" }
    }
}
```

| `type` | Format Pattern | Contoh |
|--------|---------------|--------|
| `number` | Excel number format string | `#,##0.00`, `0%` |
| `date` | Java date pattern | `dd/MM/yyyy`, `MMM yyyy` |
| `boolean` | Tidak relevan (Excel native) | - |
| `text` | Tidak relevan | - |

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
