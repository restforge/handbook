# Query List dan Read (List and Read Queries)

> `datatablesQuery`, `viewQuery`, dan `viewName` untuk endpoint `/datatables`, `/read`, `/first`, dan `/lookup`.

Kembali ke [ikhtisar Query Declarative](README.md).

Tiga properti ini menentukan sumber data untuk operasi baca. `datatablesQuery` melayani list dengan paginasi, sedangkan `viewName` dan `viewQuery` menentukan sumber data untuk read satu record dan lookup dropdown.

---

## Menyiapkan Query Datatables (Datatables Query)

`datatablesQuery` menjadi query utama untuk endpoint `/datatables` dengan paginasi dan pencarian. Properti ini wajib ada di setiap payload.

```json
{
  "tableName": "category",
  "primaryKey": "category_id",
  "fieldName": ["category_id", "category_name", "is_active"],
  "datatablesQuery": "select category_id, category_name, is_active from category",
  "action": {
    "datatables": true,
    "read": true,
    "first": true,
    "create": true,
    "update": true,
    "delete": true
  }
}
```

Pada tabel sederhana tanpa relasi, cukup definisikan `datatablesQuery`. Endpoint `/read` dan `/first` otomatis memakai `tableName` langsung, dan hasilnya tetap konsisten karena response difilter berdasarkan `fieldName`.

| Endpoint | Query yang dipakai |
|----------|--------------------|
| `/datatables` | `SELECT category_id, category_name, is_active FROM category` |
| `/read` | `SELECT * FROM category` (fallback ke `tableName`) |
| `/first` | `SELECT * FROM category WHERE ...` (fallback ke `tableName`) |

> **Catatan:** Alias tabel (`a`, `b`, dan seterusnya) direkomendasikan saat query mengandung JOIN untuk menghindari ambiguitas kolom. `datatablesQuery` mendukung deklarasi inline maupun file reference, lihat [Query sebagai File](file-reference.md).

---

## Menampilkan Data Relasi di Read lewat Database VIEW (View Name)

`viewName` menunjuk ke database VIEW yang sudah dibuat. Apabila didefinisikan, endpoint `/read`, `/first`, `/lookup`, dan header `/read-composite` membaca data dari VIEW tersebut, bukan dari tabel utama.

Buat VIEW di database terlebih dahulu, lalu daftarkan namanya di payload.

```sql
create or replace view v_item_product as
select a.item_product_id, a.product_code, a.sku, a.product_name,
       a.category_id, b.category_name, a.selling_price, a.stock,
       a.uom, a.is_active
from item_product a
inner join category b on b.category_id = a.category_id;
```

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
  "action": {
    "datatables": true, "read": true, "first": true, "lookup": true,
    "create": true, "update": true, "delete": true, "export": true, "import": true
  }
}
```

Endpoint `/first` dan `/lookup` ikut memperoleh `category_name` karena membaca dari VIEW, sehingga data relasi konsisten di seluruh operasi read.

| Endpoint | Query yang dipakai |
|----------|--------------------|
| `/datatables` | `datatablesQuery` (inline JOIN) |
| `/read` | `SELECT * FROM v_item_product` (prioritas `viewName`) |
| `/first` | `SELECT * FROM v_item_product WHERE ...` |
| `/lookup` | `SELECT ... FROM v_item_product ...` |

> **Catatan:** Operasi write (insert, update, delete) tetap memakai `tableName`, bukan VIEW. `viewName` hanya memengaruhi `readSource`.

---

## Menampilkan Data Relasi tanpa VIEW (View Query)

`viewQuery` berfungsi sebagai virtual view yang menggantikan kebutuhan membuat VIEW di database. Properti ini melayani endpoint read yang sama dengan `viewName`, yaitu `/read`, `/first`, dan `/lookup`, sehingga cocok ketika akses DDL terbatas atau pembuatan VIEW tidak diperbolehkan oleh kebijakan DBA.

```json
{
  "tableName": "item_product",
  "primaryKey": "item_product_id",
  "fieldName": [
    "item_product_id", "product_code", "sku", "product_name",
    "category_id", "category_name", "selling_price", "stock",
    "uom", "is_active"
  ],
  "datatablesQuery": "select a.item_product_id, a.product_code, a.sku, a.product_name, b.category_name, a.selling_price, a.stock, a.uom, a.is_active, a.category_id from item_product a inner join category b on b.category_id = a.category_id",
  "viewQuery": "select a.item_product_id, a.product_code, a.sku, a.product_name, a.category_id, b.category_name, a.selling_price, a.stock, a.uom, a.is_active from item_product a inner join category b on b.category_id = a.category_id",
  "action": {
    "datatables": true, "read": true, "first": true, "lookup": true,
    "create": true, "update": true, "delete": true
  }
}
```

Pada `/read`, `viewQuery` dipakai langsung. Pada `/first` dan `/lookup`, `viewQuery` dibungkus sebagai subquery, misalnya `SELECT ... FROM (viewQuery) a WHERE ...`, sehingga kolom hasil JOIN tetap tersedia.

| Endpoint | Query yang dipakai |
|----------|--------------------|
| `/datatables` | `datatablesQuery` (inline JOIN) |
| `/read` | `viewQuery` |
| `/first` | `viewQuery` (dibungkus subquery) |
| `/lookup` | `viewQuery` (dibungkus subquery) |

> **Catatan:** `viewQuery` dan `viewName` melayani endpoint read yang sama (`/read`, `/first`, `/lookup`), sehingga `/first` dan `/lookup` ikut memperoleh kolom relasi seperti `category_name`. Perbedaannya hanya pada mekanisme: `viewName` memerlukan database VIEW yang sudah dibuat, sedangkan `viewQuery` adalah SQL inline. Apabila keduanya ditetapkan, `viewName` menang karena prioritasnya lebih tinggi.

---

## Membedakan Query Datatables dan Read (Different Queries)

Definisikan `datatablesQuery` dan `viewQuery` secara terpisah ketika list (`/datatables`) hanya perlu kolom ringkas, sedangkan read (`/read`) perlu kolom lengkap termasuk relasi.

```json
{
  "tableName": "sales_order",
  "primaryKey": "order_id",
  "fieldName": [
    "order_id", "order_number", "order_date", "customer_id",
    "customer_name", "total_amount", "status", "notes"
  ],
  "datatablesQuery": "select a.order_id, a.order_number, a.order_date, b.customer_name, a.total_amount, a.status from sales_order a inner join customer b on b.customer_id = a.customer_id",
  "viewQuery": "select a.order_id, a.order_number, a.order_date, a.customer_id, b.customer_name, a.total_amount, a.status, a.notes, b.phone, b.address from sales_order a inner join customer b on b.customer_id = a.customer_id",
  "action": {
    "datatables": true,
    "read": true,
    "first": true
  }
}
```

Endpoint `/datatables` mengembalikan 6 kolom ringkasan untuk tabel di UI, sedangkan `/read` mengembalikan 10 kolom detail termasuk `phone` dan `address`.

---

## Panduan Pemilihan (Decision Guide)

| Situasi | Properti yang dipakai |
|---------|------------------------|
| Tabel sederhana tanpa relasi | `datatablesQuery` saja |
| Perlu data relasi dan boleh membuat VIEW di database | `viewName` |
| Perlu data relasi tetapi tidak boleh membuat VIEW | `viewQuery` |
| List ringkas dan read lengkap berbeda kolom | `datatablesQuery` dan `viewQuery` terpisah |

---

**Lihat juga**: [Query Export](export-queries.md) · [Kolom Response dan Validasi](field-mapping.md) · [Field Sumber Data (RDF)](../../catalogs/rdf/data-source.md)
