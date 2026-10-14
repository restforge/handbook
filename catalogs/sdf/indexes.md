# Definisi Index

Single column index dideklarasikan di array `indexes`:

```javascript
indexes: ['category_code', 'is_active']
```

Composite index dideklarasikan sebagai array of array:

```javascript
indexes: [
  'category_code',
  'is_active',
  ['stock_inbound_id', 'item_product_id']
]
```

---

**Lihat juga**: [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
