# Kebijakan Inverse Relation (Inverse Relation Policy)

`dbschema-kit` menerapkan kebijakan **Inverse Opsional** untuk relasi `hasOne` dan `hasMany` di sisi parent.

## Aturan

| Tipe Relasi | Sisi Deklarasi | Status |
|-------------|----------------|--------|
| `belongsTo` | Child (yang menyimpan FK) | **Wajib** untuk menghasilkan FK constraint di DDL |
| `hasOne` | Parent (yang dirujuk) | **Opsional**, untuk dokumentasi dan tooling |
| `hasMany` | Parent (yang dirujuk) | **Opsional**, untuk dokumentasi dan tooling |

## Implikasi terhadap DDL

DDL hasil migrasi **tidak terpengaruh** apakah `hasOne`/`hasMany` di parent dideklarasikan atau tidak. Constraint FK fisik di database hanya berasal dari sisi `belongsTo` di child. Dua varian schema berikut menghasilkan DDL yang **identik**:

**Varian minimal (tanpa inverse):**

```javascript
module.exports = ({ defineModel }) => defineModel('supplier', {
  fields: {
    supplier_id:   'uuid pk',
    supplier_name: 'string:255 notnull'
  }
  // tidak ada relations
});

module.exports = ({ defineModel }) => defineModel('stock_inbound', {
  fields: {
    stock_inbound_id: 'uuid pk',
    supplier_id:      'uuid'
  },
  relations: {
    supplier: {
      type: 'belongsTo',
      localKey: 'supplier_id',
      references: 'supplier_id',
      onDelete: 'setNull'
    }
  }
});
```

**Varian lengkap (dengan inverse):**

```javascript
module.exports = ({ defineModel }) => defineModel('supplier', {
  fields: {
    supplier_id:   'uuid pk',
    supplier_name: 'string:255 notnull'
  },
  relations: {
    inbound_transactions: {
      type: 'hasMany',
      target: 'stock_inbound',
      localKey: 'supplier_id',
      references: 'supplier_id'
    }
  }
});

module.exports = ({ defineModel }) => defineModel('stock_inbound', {
  fields: {
    stock_inbound_id: 'uuid pk',
    supplier_id:      'uuid'
  },
  relations: {
    supplier: {
      type: 'belongsTo',
      localKey: 'supplier_id',
      references: 'supplier_id',
      onDelete: 'setNull'
    }
  }
});
```

Kedua varian menghasilkan tabel, kolom, dan FK constraint yang sama persis di database.

## Kapan Mendeklarasikan Inverse?

Deklarasi inverse di parent direkomendasikan jika:

1. Schema digunakan oleh tool yang butuh introspection bidirectional (misal ERD generator, payload generator)
2. File parent ingin dibuat self-explanatory (reader bisa langsung tahu siapa dependents-nya)
3. Cross-model validation diperlukan (memastikan parent dan child konsisten)

Skip deklarasi inverse jika:

1. Fokus utama hanya migration ke database
2. Keinginan menjaga schema tetap ringkas
3. Tidak ada tooling downstream yang depend pada deklarasi parent

## Validasi terkait Inverse

Validator `dbschema-kit` **tidak** akan throw error jika:

- Child punya `belongsTo` ke parent, tetapi parent tidak punya `hasOne`/`hasMany`
- Parent punya `hasMany` ke child, tetapi child tidak punya `belongsTo`

Yang akan throw error adalah:

- Parent declare `hasOne`/`hasMany` ke tabel yang tidak exist
- Field `localKey`/`references` di parent tidak exist di tabel target

---

**Lihat juga**: [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
