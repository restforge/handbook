# Field Sumber Data

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

Endpoint `/datatables` selalu memakai `datatablesQuery`, terpisah dari resolusi di atas. Endpoint `/export` memakai `exportQuery`, fallback ke `SELECT {fieldName} FROM {tableName}`.

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
| Inline SQL | String dimulai dengan `SELECT` | Diurai apa adanya |
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
| Nama kolom | Search hanya pada kolom tersebut, dipilih lewat parameter `searchBy` di request `/datatables` |
| `"all"` | Mengaktifkan pencarian lintas-kolom (search semua kolom yang terdaftar) |

Daftar ini adalah whitelist `searchBy` di runtime. Nilai `searchBy` yang tidak terdaftar di sini ditolak dengan HTTP `400`, bukan dialihkan diam-diam ke `all` seperti pada rilis sebelumnya. Parameter `search_by` milik endpoint `/read` bersifat terpisah dan divalidasi terhadap kolom yang dapat dibaca `/read`, bukan terhadap `datatablesWhere`.

Aturan penulisan entri:

- Pakai nama kolom sebagaimana muncul di hasil SELECT `datatablesQuery`, tanpa prefiks alias tabel. Entri seperti `a.supplier_code` selalu ditolak saat generate karena runtime mencocokkan nama kolom hasil SELECT.
- Setiap entri selain `all` harus cocok dengan kolom yang benar-benar tersedia bagi endpoint `/datatables`. Pengecekannya dijalankan saat generate dan dijelaskan di [Aturan `datatablesWhere`](validation-rules.md#aturan-datatableswhere).

## `viewQuery`

SQL SELECT alternatif untuk action read-only. Berguna saat resource butuh JOIN ke tabel lain untuk menampilkan kolom join (misal `category_name` di endpoint `item_product`).

```json
{
    "viewQuery": "file:query/item-product-detail.sql"
}
```

Behavior generator: `viewQuery` dibungkus sebagai subquery (`SELECT ... FROM (<viewQuery>) a`) sehingga kolom hasil JOIN bisa dipakai di WHERE clause action read-only.

## `viewName`

Nama database VIEW yang sudah ada. Prioritas lebih tinggi daripada `viewQuery`.

```json
{
    "viewName": "v_supplier_detail"
}
```

## `exportQuery`

SQL khusus untuk endpoint `/export`. Jika tidak ditetapkan, generator melakukan fallback ke `SELECT {fieldName} FROM {tableName}`, yaitu seluruh kolom yang terdaftar di `fieldName` dari tabel utama.

```json
{
    "exportQuery": "SELECT supplier_code, supplier_name, email, phone, city FROM supplier"
}
```

## Kolom Boolean di Query SQL

Kolom bertipe `boolean` di SDF disimpan dengan bentuk yang berbeda per dialect. PostgreSQL memakai tipe `BOOLEAN`, sedangkan MySQL, SQLite, dan Oracle menyimpan teks `'true'` atau `'false'`. Endpoint hasil generate menangani perbedaan ini saat insert, update, dan filter dari request, sehingga nilai `true` di request tetap cocok. SQL yang ditulis sendiri tidak ikut diubah, baik di RDF (`datatablesQuery`, `viewQuery`, `exportQuery`, `detailQuery`, `advancedQueries`, `nestedQueries`, `customQueries`) maupun di query widget dashboard.

Karena itu, pembanding boolean di query harus mengikuti dialect database seperti pada tabel berikut.

| Dialect | Pembanding yang benar | Pembanding yang salah |
|---------|-----------------------|-----------------------|
| PostgreSQL | `is_active = true` | `is_active = 1` ditolak database |
| MySQL, SQLite | `is_active = 'true'` | `is_active = true` atau `= 1` selalu bernilai salah tanpa error |
| Oracle | `is_active = 'true'` | `is_active = true` ditolak dengan `ORA-00904`, `is_active = 1` ditolak dengan `ORA-01722` |

Query berikut adalah contoh `datatablesQuery` untuk MySQL, SQLite, dan Oracle.

```sql
SELECT a.partner_id,
       a.partner_name,
       CASE WHEN a.is_active = 'true' THEN 'active' ELSE 'archived' END AS partner_status
FROM business_partner a
```

Untuk PostgreSQL, baris `CASE` ditulis dengan `a.is_active = true`. RDF yang dipindahkan ke dialect lain perlu disesuaikan pada bagian ini. [`payload validate`](../../commands/restforge-backend/payload/validate.md#saran-tindak-lanjut) menampilkan advisory bila query RDF di MySQL, SQLite, atau Oracle membandingkan kolom boolean dengan `true`, `false`, `1`, atau `0`. [`dashboard create`](../../commands/restforge-backend/dashboard/create.md#kolom-boolean-di-query-widget) menampilkan peringatan bila query widget di MySQL atau Oracle membandingkan kolom dengan `TRUE` atau `FALSE`.

## `dateTimeFields`

Deklarasi field bertipe waktu agar runtime menormalisasi input dan memformat output. Ditulis
sebagai object dengan nama field sebagai key:

```json
{
    "dateTimeFields": {
        "visit_date": { "type": "timestamp" },
        "birth_date": { "type": "date" },
        "shift_start": { "type": "time", "format": "HH:mm" }
    }
}
```

Pola `date` dan `timestamp` selalu diambil dari `DATEFORMAT` dan `DATETIMEFORMAT` aplikasi.
`format` hanya boleh ditulis untuk `type: "time"`. Aturan lengkapnya ada di
[`datetime-fields.md`](./datetime-fields.md).

## `columnFormats`

Format tampilan kolom untuk endpoint `/export` (Excel). Tipe dan pattern dapat ditetapkan per kolom.

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
