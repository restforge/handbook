# Contoh Lengkap

## Tabel Master Sederhana

```javascript
module.exports = ({ defineModel }) => defineModel('category', {
  fields: {
    category_id:   'uuid pk',
    category_code: 'string:20 unique notnull',
    category_name: 'string:100 notnull',
    description:   'string:500',
    is_active:     'boolean default:true',
    created_at:    'timestamp',
    created_by:    'string:70',
    updated_at:    'timestamp',
    updated_by:    'string:70'
  },
  indexes: ['category_code', 'is_active']
});
```

## Tabel Master dengan Foreign Key

```javascript
module.exports = ({ defineModel }) => defineModel('item_product', {
  fields: {
    item_product_id: 'uuid pk',
    product_code:    'string:20 unique notnull',
    sku:             'string:20 unique notnull',
    product_name:    'string:100 notnull',
    category_id:     'uuid notnull',
    brand:           'string:50',
    description:     'string:500',
    purchase_price:  'decimal:15,2 notnull default:0',
    selling_price:   'decimal:15,2 notnull default:0',
    stock:           'decimal:15,0 notnull default:0',
    min_stock:       'decimal:15,0 default:0',
    uom:             "string:10 default:'pcs'",
    weight:          'decimal:10,2 default:0',
    is_active:       'boolean default:true',
    show_in_store:   'boolean default:true',
    barcode:         'string:20',
    shelf_location:  'string:20',
    notes:           'string:500',
    created_at:      'timestamp',
    created_by:      'string:70',
    updated_at:      'timestamp',
    updated_by:      'string:70'
  },
  indexes: [
    'product_code',
    'sku',
    'category_id',
    'is_active',
    'barcode'
  ],
  relations: {
    category: {
      type: 'belongsTo',
      localKey: 'category_id',
      references: 'category_id',
      onDelete: 'restrict'
    }
  }
});
```

## Tabel Detail Transaksi dengan Composite Unique

```javascript
module.exports = ({ defineModel }) => defineModel('stock_inbound_item', {
  fields: {
    stock_inbound_item_id: 'uuid pk',
    stock_inbound_id:      'uuid notnull',
    line_number:           'integer notnull',
    item_product_id:       'uuid notnull',
    qty_received:          'decimal:15,0 notnull default:0',
    uom:                   "string:10 default:'pcs'",
    unit_price:            'decimal:15,2 default:0',
    total_amount:          'decimal:15,2 default:0',
    notes:                 'text',
    created_at:            'timestamp',
    created_by:            'string:70',
    updated_at:            'timestamp',
    updated_by:            'string:70'
  },
  uniques: [
    ['stock_inbound_id', 'line_number']
  ],
  indexes: [
    'stock_inbound_id',
    'item_product_id',
    ['stock_inbound_id', 'item_product_id']
  ],
  relations: {
    header: {
      type: 'belongsTo',
      target: 'stock_inbound',
      localKey: 'stock_inbound_id',
      references: 'stock_inbound_id',
      onDelete: 'cascade'
    },
    product: {
      type: 'belongsTo',
      target: 'item_product',
      localKey: 'item_product_id',
      references: 'item_product_id',
      onDelete: 'restrict'
    }
  }
});
```

## Absensi Lintas Zona

Tabel absensi karyawan yang mencatat momen clock-in dan clock-out. Field waktu
bertipe `timestamptz` agar momen yang tersimpan tetap benar meski karyawan berada
di kota dengan zona berbeda. Jadwal shift memakai `time` karena hanya jam tanpa
tanggal yang dicatat.

```javascript
module.exports = ({ defineModel }) => defineModel('attendance', {
  fields: {
    attendance_id: 'uuid pk',
    employee_id:   'string:20 notnull',
    shift_start:   "time notnull default:'08:00:00'",
    clockin:       'timestamptz notnull',
    clockout:      'timestamptz',
    notes:         'string:200',
    created_at:    'timestamptz default:now()',
    created_by:    'string:70'
  },
  indexes: ['employee_id', 'clockin']
});
```

Karyawan di Surabaya (`Asia/Jakarta`, UTC+7) dan Balikpapan (`Asia/Makassar`,
UTC+8) sama-sama clock-in pukul 07.00 waktu setempat. Masing-masing mengirim ISO
dengan offset lokalnya sendiri:

```json
{"employee_id":"EMP-001","clockin":"2026-09-18T07:00:00+07:00"}
{"employee_id":"EMP-002","clockin":"2026-09-18T07:00:00+08:00"}
```

Offset eksplisit pada kedua request menentukan momennya, sehingga tersimpan
sebagai dua momen berbeda meski jam lokalnya sama-sama pukul 07.00: karyawan
Balikpapan clock-in satu jam lebih dahulu secara absolut. Query baca tidak
memerlukan join ke tabel zona atau kolom timezone tambahan; response
mengembalikan momen sebagai ISO UTC:

```json
{
  "data": [
    {"employee_id":"EMP-001","clockin":"2026-09-18T00:00:00.000Z"},
    {"employee_id":"EMP-002","clockin":"2026-09-17T23:00:00.000Z"}
  ]
}
```

Frontend masing-masing kota mengonversi nilai ISO tersebut ke zona lokal
komputer pembaca sebelum ditampilkan, sehingga tetap terlihat pukul 07.00 di
komputer asal. Detail perilaku konfigurasi zona dan format ada di
[`rdf/datetime-fields.md`](../rdf/datetime-fields.md#konfigurasi-zona-dan-format).

---

**Lihat juga**: [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
