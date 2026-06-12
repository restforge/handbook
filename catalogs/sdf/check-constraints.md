# Definisi CHECK Constraint

CHECK constraint dideklarasikan di property `checks` di level options. Mendukung dua kategori operator:

1. **Enum check** (`in`): membatasi nilai pada list yang ditentukan
2. **Comparison check** (`gt`, `gte`, `lt`, `lte`, `eq`, `neq`): membatasi nilai numeric atau scalar

## Sintaks `checks`

```javascript
checks: [
  { field: <fieldName>, <operator>: <value> },
  ...
]
```

| Operator | Tipe value | SQL output |
|----------|------------|-----------|
| `in` | array of literal | `field IN (<value1>, <value2>, ...)` |
| `gt` | number | `field > <value>` |
| `gte` | number | `field >= <value>` |
| `lt` | number | `field < <value>` |
| `lte` | number | `field <= <value>` |
| `eq` | scalar | `field = <value>` |
| `neq` | scalar | `field != <value>` |

## Aturan

1. `field` wajib, harus ada di property `fields`
2. Tepat satu operator key per entry. Bila lebih dari satu operator ditulis, validator akan melempar error
3. Bila butuh multiple constraint pada field yang sama (misal `unit_price >= 0 AND unit_price <= 1000000`), tulis dua entry terpisah
4. Constraint name auto-generated: `chk_<table>_<field>`. Bila ada duplikat field, nama diberi suffix operator: `chk_<table>_<field>_<op>`. Limit 30 karakter berlaku (Oracle compatibility)
5. Untuk override nama constraint, gunakan property `name`:

```javascript
checks: [
  { name: 'chk_status_enum', field: 'status', in: ['draft', 'confirmed'] }
]
```

## Contoh Penggunaan

```javascript
module.exports = ({ defineModel }) => defineModel('stock_inbound', {
  fields: {
    status:     "string:20 default:'draft'",
    total_qty:  'decimal:15,0 default:0'
  },
  checks: [
    { field: 'status', in: ['draft', 'confirmed', 'closed', 'cancelled'] },
    { field: 'total_qty', gte: 0 }
  ]
});
```

```javascript
module.exports = ({ defineModel }) => defineModel('stock_inbound_item', {
  fields: {
    line_number: 'integer notnull',
    unit_price:  'decimal:15,2 default:0'
  },
  checks: [
    { field: 'line_number', gt: 0 },
    { field: 'unit_price', gte: 0 }
  ]
});
```

## Lingkup yang TIDAK Dicakup

- Composite check antar field (misal `start_date < end_date`)
- Compound check dengan AND/OR/NOT
- Pattern check (LIKE, REGEXP)
- Function-based check (misal `LENGTH(field) > 10`)

---

**Lihat juga**: [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
