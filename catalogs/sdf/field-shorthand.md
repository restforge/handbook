# Sintaks Field Shorthand (Field Shorthand Syntax)

Setiap field dideklarasikan sebagai pasangan key-value:

```javascript
fields: {
  nama_field: 'type[:modifier] [constraint1] [constraint2] ...'
}
```

Aturan format string shorthand:

1. Token pertama selalu type (wajib)
2. Type bisa diikuti modifier dengan separator titik dua (misal `string:20`)
3. Token berikutnya adalah constraint, dipisah spasi
4. Constraint bisa standalone (misal `notnull`) atau punya value dengan separator titik dua (misal `default:true`)
5. Urutan constraint tidak signifikan

**Contoh:**

```javascript
fields: {
  category_id:   'uuid pk',
  category_code: 'string:20 unique notnull',
  description:   'string:500',
  is_active:     'boolean default:true',
  created_at:    'timestamp'
}
```

---

**Lihat juga**: [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
