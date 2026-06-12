# Master-Detail Composite (Composite Read)

> `detailQuery` untuk mengambil detail items pada endpoint `/read-composite`.

Kembali ke [ikhtisar Query Declarative](README.md).

`detailQuery` menentukan query yang mengambil detail items berdasarkan foreign key dari header. Properti ini terletak di dalam `masterDetail.detailConfig`, bukan di root payload. Struktur `masterDetail` lengkap dijelaskan di referensi [`catalogs/rdf/master-detail.md`](../../catalogs/rdf/master-detail.md).

> **Prasyarat:** `masterDetail.enabled: true` dan `action.readComposite: true`. Tanpa keduanya, endpoint `/read-composite` tidak di-generate.

---

## Memahami Resolusi Composite Read (Composite Resolution)

Endpoint `/read-composite` mengembalikan header beserta detail items dalam satu response. Resolusi query dilakukan pada dua level yang independen.

| Level | Sumber query | Fallback |
|-------|--------------|----------|
| Header | Sama dengan `/read`: `viewName` lalu `viewQuery` lalu `tableName` | `SELECT * FROM tableName` |
| Detail | `masterDetail.detailConfig.detailQuery` | `SELECT * FROM {detailTable} WHERE {foreignKey} = ? ORDER BY line_number` |

Seluruh setup query declarative untuk `/read` otomatis berlaku pada header `/read-composite`, sehingga header dapat dikombinasikan dengan `viewName` atau `viewQuery` apabila perlu JOIN.

---

## Mengambil Detail dengan JOIN (Detail Query with JOIN)

Definisikan `detailQuery` ketika detail items perlu JOIN ke tabel lain, misalnya mengambil `product_code` dan `product_name` dari `item_product`.

```json
{
  "tableName": "stock_inbound",
  "primaryKey": "stock_inbound_id",
  "fieldName": [
    "stock_inbound_id", "inbound_number", "inbound_date",
    "warehouse_id", "supplier_id", "reference_number",
    "notes", "total_items", "total_qty", "total_amount", "status"
  ],
  "datatablesQuery": "file:query/stock-inbound-datatables.sql",
  "action": {
    "datatables": true, "create": true, "update": true, "delete": true,
    "read": true, "createComposite": true, "updateComposite": true, "readComposite": true
  },
  "masterDetail": {
    "enabled": true,
    "detailTable": "stock_inbound_item",
    "foreignKey": "stock_inbound_id",
    "detailConfig": {
      "tableName": "stock_inbound_item",
      "primaryKey": "stock_inbound_item_id",
      "fieldName": [
        "stock_inbound_item_id", "stock_inbound_id", "line_number",
        "item_product_id", "qty_received", "uom",
        "unit_price", "total_amount", "notes"
      ],
      "detailQuery": "file:query/stock-inbound-detailquery.sql"
    }
  }
}
```

File `query/stock-inbound-detailquery.sql` (PostgreSQL):

```sql
select a.stock_inbound_item_id,
       a.stock_inbound_id,
       a.line_number,
       a.item_product_id,
       b.product_code,
       b.product_name,
       a.qty_received,
       a.uom,
       a.unit_price,
       a.total_amount,
       a.notes
from stock_inbound_item a
inner join item_product b on b.item_product_id = a.item_product_id
where a.stock_inbound_id = $1
order by a.line_number
```

Pada response, header memakai `tableName` (atau `viewName`/`viewQuery` apabila ditetapkan), sedangkan detail memakai `detailQuery` yang melakukan JOIN ke `item_product`.

> **Catatan:** Apabila detail cukup `SELECT * FROM detailTable WHERE foreignKey = ? ORDER BY line_number`, `detailQuery` tidak perlu didefinisikan. Fallback otomatis sudah memadai.

---

## Menulis Placeholder Foreign Key (Foreign Key Placeholder)

`detailQuery` harus memuat satu placeholder untuk nilai foreign key header. Bentuk placeholder mengikuti sintaks native database target.

| Database | Placeholder | Contoh |
|----------|-------------|--------|
| PostgreSQL | `$1` | `... WHERE stock_inbound_id = $1` |
| MySQL | `?` | `... WHERE stock_inbound_id = ?` |
| Oracle | `:1` | `... WHERE stock_inbound_id = :1` |

Saat `/read-composite` dipanggil, placeholder diganti dengan nilai primary key header pada setiap iterasi detail.

> **Catatan:** Tulis placeholder sesuai database target. Untuk detail yang ditulis sebagai file reference, isi file SQL harus memakai placeholder native database tujuan, lihat [Query sebagai File](file-reference.md).

---

## Proses Generation (Generation Process)

`detailQuery` diproses sepenuhnya saat generation, bukan saat runtime.

1. Prefix `file:` di-expand oleh payload-processor saat `endpoint create` dijalankan, sehingga konten SQL dibaca dari file pada tahap generation.
2. Konten SQL disisipkan sebagai template literal di dalam method `readComposite()` pada module `.js` yang di-generate.
3. Kode runtime hasil generate mengeksekusi SQL tersebut langsung tanpa akses filesystem.

> **Catatan:** Generator memvalidasi bahwa tidak ada prefix `file:` yang tersisa setelah tahap expand. Karena SQL sudah inline di module, tidak ada pembacaan file SQL saat runtime. Mekanisme `file:` yang berlaku untuk seluruh field query dijelaskan di referensi [`catalogs/rdf/file-reference.md`](../../catalogs/rdf/file-reference.md).

---

## Kapan Memakai Detail Query (When to Use)

| Kebutuhan | Pakai `detailQuery` |
|-----------|---------------------|
| Detail perlu JOIN ke tabel lain | Ya |
| Kolom detail berbeda dari kolom fisik tabel detail | Ya |
| Memerlukan urutan khusus selain `line_number` | Ya |
| Ingin menyimpan query detail sebagai file terpisah | Ya |
| Detail cukup ambil seluruh kolom apa adanya | Tidak perlu, fallback sudah memadai |

---

**Lihat juga**: [Field Master Detail (RDF)](../../catalogs/rdf/master-detail.md) · [Query sebagai File](file-reference.md) · [Query List dan Read](read-queries.md)
