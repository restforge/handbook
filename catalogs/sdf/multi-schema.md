# Schema Namespace (Multi-Schema Support)

Untuk database yang mendukung multi-schema (PostgreSQL, Oracle, SQL Server), property `schema` di level options dapat digunakan untuk menempatkan tabel di namespace tertentu.

```javascript
module.exports = ({ defineModel }) => defineModel('products', {
  schema: 'inventory',
  fields: {
    product_id:   'uuid pk',
    product_code: 'string:20 unique notnull',
    product_name: 'string:255 notnull'
  }
});
```

Aturan:

| Aturan | Penjelasan |
|--------|-----------|
| Default value | Bila `schema` tidak di-set, tabel masuk ke default schema (PostgreSQL: `public`, Oracle: schema user, MySQL: database aktif) |
| Format value | Snake_case, sama dengan konvensi `tableName` |
| Cross-schema FK | Relasi `belongsTo` ke tabel di schema lain didukung. Reference `target` bisa fully-qualified (`inventory.products`) atau bare (`products` jika schema yang sama) |
| MySQL & SQLite | Property `schema` diabaikan (kedua dialect tidak punya konsep schema namespace yang setara) |
| Folder layout introspect | Saat introspect database multi-schema, output file di-organize per schema dalam subfolder `<output>/<schema>/<table>.js` |

**Cross-schema reference contoh:**

```javascript
module.exports = ({ defineModel }) => defineModel('order_item', {
  schema: 'sales',
  fields: {
    order_item_id: 'uuid pk',
    product_id:    'uuid notnull'
  },
  relations: {
    product: {
      type: 'belongsTo',
      target: 'inventory.products',  // qualified reference
      localKey: 'product_id',
      references: 'product_id'
    }
  }
});
```

---

**Lihat juga**: [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
