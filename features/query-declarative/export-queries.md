# Query Export (Export Query)

> `exportQuery` untuk menentukan kolom dan format data yang diekspor ke file Excel.

Kembali ke [ikhtisar Query Declarative](README.md).

`exportQuery` menentukan data yang diekspor oleh endpoint `/export`. Kolom yang di-select pada query inilah yang muncul di file Excel, sehingga export dapat memuat alias dan transformasi yang berbeda dari tampilan UI.

---

## Menyiapkan Query Export (Setting Up Export)

Definisikan `exportQuery` dan aktifkan `action.export: true`. Query export boleh memuat alias kolom dan transformasi nilai di level SQL.

```json
{
  "tableName": "item_product",
  "primaryKey": "item_product_id",
  "viewName": "v_item_product",
  "fieldName": [
    "item_product_id", "product_code", "sku", "product_name",
    "category_id", "category_name", "selling_price", "stock",
    "uom", "is_active"
  ],
  "datatablesQuery": "select a.item_product_id, a.product_code, a.sku, a.product_name, b.category_name, a.selling_price, a.stock, a.uom, a.is_active, a.category_id from item_product a inner join category b on b.category_id = a.category_id",
  "exportQuery": "file:query/item_product_export.sql",
  "columnFormats": {
    "selling_price": { "type": "number", "format": "#,##0.00" },
    "stock": { "type": "number", "format": "#,##0" },
    "is_active": { "type": "boolean" }
  },
  "action": {
    "datatables": true,
    "read": true,
    "export": true
  }
}
```

File `query/item_product_export.sql`:

```sql
select a.product_code as "Kode Produk",
       a.product_name as "Nama Produk",
       b.category_name as "Kategori",
       a.selling_price as "Harga Jual",
       a.stock as "Stok",
       a.uom as "Satuan",
       case when a.is_active then 'Aktif' else 'Tidak Aktif' end as "Status"
from item_product a
inner join category b on b.category_id = a.category_id
order by b.category_name, a.product_name
```

Hasilnya, file Excel memuat kolom dengan header bahasa Indonesia dan nilai `is_active` yang sudah ditransformasi menjadi teks `Aktif` atau `Tidak Aktif`, sementara `/datatables` tetap menampilkan kolom teknis sesuai `datatablesQuery`.

> **Catatan:** Apabila `exportQuery` tidak didefinisikan, endpoint `/export` memakai fallback `SELECT {fieldName} FROM {tableName}`, yaitu seluruh kolom yang terdaftar di `fieldName` dari tabel utama.

---

## Menggunakan File Reference untuk Export (Export as File)

Query export yang panjang lebih mudah dikelola sebagai file terpisah dengan prefix `file:`.

```json
{
  "exportQuery": "file:query/item_product_export.sql"
}
```

Penjelasan format `file:`, struktur folder, dan proses generation ada di [Query sebagai File](file-reference.md).

---

## Kapan Memakai Export Query (When to Use)

| Kebutuhan | Pakai `exportQuery` |
|-----------|---------------------|
| Kolom export berbeda dari kolom UI | Ya |
| Memerlukan alias kolom berbahasa Indonesia | Ya |
| Memerlukan transformasi nilai (boolean ke teks, format angka) | Ya |
| Memerlukan urutan `ORDER BY` khusus untuk Excel | Ya |
| Export cukup seluruh kolom apa adanya | Tidak perlu, fallback sudah memadai |

---

**Lihat juga**: [Query List dan Read](read-queries.md) · [Query sebagai File](file-reference.md) · [Field Sumber Data (RDF)](../../catalogs/rdf/data-source.md)
