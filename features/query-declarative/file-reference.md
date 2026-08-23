# Query sebagai File (Query as File)

> Menyimpan SQL sebagai file terpisah dengan prefix `file:` agar payload tetap ringkas.

Kembali ke [ikhtisar Query Declarative](README.md).

Format `file:` berfungsi untuk memindahkan SQL panjang dari payload JSON ke file `.sql` tersendiri. Generator membaca konten file pada saat generation dan menyisipkannya sebagai SQL inline di module hasil generate. Spec field RDF yang mendukung `file:` ada di referensi [`catalogs/rdf/file-reference.md`](../../catalogs/rdf/file-reference.md).

---

## Menulis File Reference (Writing a File Reference)

Ganti nilai SQL inline dengan string berprefix `file:` yang menunjuk ke path relatif file SQL.

```json
{
  "datatablesQuery": "file:query/item_product_list.sql"
}
```

Aturan path:

| Aturan | Keterangan |
|--------|------------|
| Basis path | Relatif terhadap lokasi file payload JSON |
| Ekstensi | Bebas, `.sql` direkomendasikan |
| Isi file | Satu statement SQL SELECT |
| Waktu resolusi | Saat generation, bukan saat runtime |

---

## Memindahkan Semua Query ke File (All Queries as Files)

Seluruh properti query yang berisi SQL mendukung `file:`, yaitu `datatablesQuery`, `viewQuery`, `exportQuery`, dan `detailQuery`. Properti `viewName` tidak mendukung `file:` karena berisi nama database object, bukan SQL.

```json
{
  "tableName": "item_product",
  "primaryKey": "item_product_id",
  "fieldName": [
    "item_product_id", "product_code", "sku", "product_name",
    "category_id", "category_name", "selling_price", "stock",
    "uom", "is_active"
  ],
  "datatablesQuery": "file:query/item_product_list.sql",
  "viewQuery": "file:query/item_product_view.sql",
  "exportQuery": "file:query/item_product_export.sql",
  "action": {
    "datatables": true,
    "read": true,
    "first": true,
    "export": true,
    "import": true
  }
}
```

Pendekatan ini menjaga payload tetap ringkas, memudahkan pengujian query secara terpisah di SQL client, dan memungkinkan file SQL dipakai kembali oleh payload lain dengan query yang sama.

---

## Menyusun Struktur Folder (Folder Structure)

Letakkan seluruh file SQL di subfolder `query/` di bawah folder payload agar mudah dikelola.

```
payload/
├── item-product.json
├── supplier.json
├── category.json
├── stock-inbound.json
└── query/
    ├── item_product.sql
    ├── item_product_list.sql
    ├── item_product_export.sql
    ├── supplier_list.sql
    ├── stock-inbound-datatables.sql
    └── stock-inbound-detailquery.sql
```

---

## Memahami Proses Generation (Generation Process)

Konten file SQL diproses saat generation, bukan saat runtime.

1. Prefix `file:` di-expand oleh payload-processor saat `endpoint create` dijalankan.
2. Konten SQL disisipkan sebagai template literal di dalam module `.js` yang di-generate.
3. Module runtime mengeksekusi SQL inline tersebut tanpa membaca file SQL kembali.

> **Catatan:** Karena SQL sudah inline di module, tidak ada akses filesystem dari kode runtime hasil generate. Generator juga memvalidasi bahwa tidak ada prefix `file:` yang tersisa setelah tahap expand, sehingga file yang hilang akan terdeteksi pada saat generation.

---

## Menulis Placeholder pada File SQL (Placeholder in File)

Pada file SQL untuk `detailQuery`, tulis placeholder foreign key sesuai sintaks native database target.

| Database | Placeholder |
|----------|-------------|
| PostgreSQL | `$1` |
| MySQL | `?` |
| Oracle | `:1` |

```sql
select a.stock_inbound_item_id, a.line_number, a.item_product_id,
       b.product_code, b.product_name, a.qty_received, a.unit_price, a.total_amount
from stock_inbound_item a
inner join item_product b on b.item_product_id = a.item_product_id
where a.stock_inbound_id = $1
order by a.line_number
```

> **Catatan:** Penjelasan placeholder per database dan resolusi composite read ada di [Master-Detail Composite](master-detail.md).

---

**Lihat juga**: [Format Referensi File (RDF)](../../catalogs/rdf/file-reference.md) · [Master-Detail Composite](master-detail.md) · [Query List dan Read](read-queries.md)
