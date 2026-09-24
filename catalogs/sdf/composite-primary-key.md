# Definisi Composite Primary Key

Untuk tabel dengan composite primary key, gunakan property `primaryKey`:

```javascript
module.exports = ({ defineModel }) => defineModel('inventory_balance', {
  fields: {
    item_product_id: 'uuid',
    warehouse_id:    'uuid',
    period_date:     'date',
    qty_balance:     'decimal:15,0 default:0'
  },
  primaryKey: ['item_product_id', 'warehouse_id', 'period_date']
});
```

Aturan terkait composite primary key:

1. Field individual tidak menggunakan constraint `pk` ketika composite primary key digunakan
2. Semua field yang tercantum di array `primaryKey` otomatis dianggap `notnull` tanpa perlu ditulis eksplisit
3. Field di `primaryKey` harus ada di property `fields`, jika tidak, validator akan melempar error

---

**Lihat juga**: [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
