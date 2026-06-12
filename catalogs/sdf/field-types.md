# Sistem Tipe

Tipe bersifat logical, bukan physical. DDL generator akan memetakan ke tipe SQL native sesuai dialect target.

| Shorthand | Modifier | Deskripsi |
|-----------|----------|-----------|
| `string:N` | length wajib | String dengan panjang maksimum N karakter |
| `text` | tidak ada | String panjang tanpa batas spesifik |
| `integer` | tidak ada | Bilangan bulat ukuran standar |
| `bigint` | tidak ada | Bilangan bulat ukuran besar |
| `decimal:M,N` | precision, scale wajib | Bilangan desimal presisi tetap |
| `boolean` | tidak ada | Nilai benar/salah |
| `date` | tidak ada | Tanggal tanpa waktu |
| `timestamp` | tidak ada | Tanggal dengan waktu |
| `uuid` | tidak ada | Universally unique identifier |
| `json` | tidak ada | Struktur data JSON |

**Contoh penggunaan:**

```javascript
fields: {
  product_code:   'string:20',
  description:    'text',
  stock_qty:      'integer',
  total_amount:   'decimal:15,2',
  is_active:      'boolean',
  register_date:  'date',
  created_at:     'timestamp',
  product_id:     'uuid',
  metadata:       'json'
}
```

---

**Lihat juga**: [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
