# Sistem Constraint

Constraint mengikuti type, dipisah spasi, urutan bebas.

| Shorthand | Format | Deskripsi |
|-----------|--------|-----------|
| `pk` | standalone | Menandai field sebagai primary key |
| `notnull` | standalone | Field wajib diisi (tidak boleh NULL) |
| `unique` | standalone | Nilai harus unik di seluruh tabel |
| `default:VALUE` | dengan value | Nilai default saat insert tanpa nilai |
| `fk:TABLE.COLUMN` | dengan value | Foreign key ke kolom tabel lain |
| `index` | standalone | Auto-create single column index |

> **Catatan**: Constraint `autoUpdate` **DEPRECATED** dan tidak termasuk di tabel di atas. Engine masih menerima token ini untuk backward compatibility, namun token tersebut tidak memiliki efek fungsional apa pun (output DDL identik dengan `'timestamp'` biasa). Auto-update untuk field `updated_at` ditangani sepenuhnya oleh layer RDF (konvensi `auditColumns` di BaseModel runtime). Dokumentasi historis dapat dibaca di bagian [Constraint `autoUpdate` (DEPRECATED)](#constraint-autoupdate-deprecated) di bawah, namun **tidak boleh digunakan di template baru**.

## Constraint `pk`

Menandai field sebagai primary key. Hanya satu field per model yang boleh punya constraint `pk`.

```javascript
category_id: 'uuid pk'
```

Untuk composite primary key, gunakan property `primaryKey` di level options (lihat halaman [`composite-primary-key.md`](./composite-primary-key.md)).

## Constraint `notnull`

Field wajib diisi.

```javascript
category_name: 'string:100 notnull'
```

Jika field punya `pk`, otomatis `notnull` (tidak perlu ditulis eksplisit). Aturan auto-`notnull` ini juga berlaku untuk semua field yang tercantum di property `primaryKey` (composite primary key).

## Constraint `unique`

Nilai harus unik. Untuk composite unique, gunakan property `uniques` di level options.

```javascript
category_code: 'string:20 unique'
```

## Constraint `default:VALUE`

Nilai default saat insert. Format value:

| Tipe value | Sintaks shorthand | Hasil |
|------------|-------------------|-------|
| Boolean | `default:true` atau `default:false` | Boolean literal |
| Integer | `default:0`, `default:100` | Integer literal |
| String | `default:'INDONESIA'` | String literal (gunakan single quote) |
| Function | `default:now()` | Pemanggilan function (di-translate ke native function per dialect) |
| Constant | `default:current_date` | SQL constant |

### Translasi `default:now()` per Dialect

Function `now()` diterjemahkan oleh DDL generator ke native function per dialect:

| Dialect | Native function |
|---------|-----------------|
| PostgreSQL | `CURRENT_TIMESTAMP` |
| MySQL | `NOW()` |
| Oracle | `SYSTIMESTAMP` |
| SQLite | `CURRENT_TIMESTAMP` |

**Contoh:**

```javascript
fields: {
  is_active:      'boolean default:true',
  amount_payable: 'decimal:15,2 default:0',
  country:        'string:100 notnull default:\'INDONESIA\'',
  inbound_date:   'date notnull default:current_date',
  created_at:     'timestamp default:now()'
}
```

## Constraint `fk:TABLE.COLUMN`

Mendeklarasikan foreign key dengan referensi ke kolom tabel lain.

```javascript
fields: {
  category_id: 'uuid fk:category.category_id notnull'
}
```

Untuk konfigurasi advanced (ON DELETE, ON UPDATE), gunakan property `relations` di level options (lihat halaman [`foreign-keys.md`](./foreign-keys.md)).

## Constraint `index`

Auto-create single column index pada field tersebut. Alternatif lebih singkat daripada mendaftarkan field di array `indexes`.

```javascript
fields: {
  inbound_date: 'date notnull index'
}
```

Setara dengan:

```javascript
fields: {
  inbound_date: 'date notnull'
},
indexes: ['inbound_date']
```

## Constraint `autoUpdate` (DEPRECATED)

> **STATUS: DEPRECATED.** Constraint ini tidak boleh digunakan di template baru. Engine masih menerima token ini untuk backward compatibility, namun token tersebut tidak memiliki efek fungsional apa pun.

### Konvensi Baru (Yang Harus Digunakan)

Untuk field `updated_at`, gunakan deklarasi plain:

```javascript
fields: {
  updated_at: 'timestamp'
}
```

Auto-update untuk field `updated_at` ditangani sepenuhnya oleh **layer RDF** (Runtime Definition Format) melalui konvensi `auditColumns` di BaseModel runtime. Mekanisme ini bekerja berdasarkan **konvensi nama field**, bukan berdasarkan marker SDF. Detail penanganan dapat dibaca di [`rdf/audit-columns.md`](../rdf/audit-columns.md).

### Keterbatasan Modifier `autoUpdate` (Mengapa Di-deprecate)

Investigasi source code menunjukkan modifier ini secara fungsional tidak memberikan kontribusi apa pun:

- **Output DDL identik** dengan `'timestamp'` biasa. Tidak ada `ON UPDATE` clause, tidak ada trigger, tidak ada DEFAULT.
- **Tidak terdeteksi oleh diff-engine** sehingga penambahan/penghapusan marker tidak menghasilkan migration script.
- **Tidak dapat dipulihkan** dari introspeksi database (roundtrip SDF → DB → SDF akan kehilangan marker).
- **Tidak ada consumer aktif** di codebase yang membaca flag `field.autoUpdate` dari IR untuk menghasilkan behavior berbeda. Runtime BaseModel menyisipkan `updated_at = CURRENT_TIMESTAMP` berdasarkan nama kolom audit standar, bukan berdasarkan marker.

### Migrasi dari Pemakaian Lama

| Sebelum | Sesudah |
|---------|---------|
| `updated_at: 'timestamp autoUpdate'` | `updated_at: 'timestamp'` |

Tidak ada fungsionalitas yang hilang karena auto-update tetap bekerja di runtime tanpa marker tersebut.

### Aturan Engine yang Masih Berlaku (Backward Compatibility)

Untuk template lama yang masih menggunakan modifier ini, engine masih memberlakukan aturan validasi berikut:

- Hanya valid untuk type `timestamp` dan `date`. Type lain akan melempar error.
- Tidak boleh dikombinasikan dengan `default:` shorthand (misal `'timestamp default:now() autoUpdate'` akan melempar error).

## Auto-Index untuk `pk` dan `fk`

Generator akan otomatis menghasilkan index untuk:

| Sumber | Behavior |
|--------|----------|
| Field dengan `pk` | Index dibuat secara implisit oleh DDL constraint primary key (semua dialect) |
| Field dengan `fk:` shorthand | Index eksplisit dibuat dengan nama `idx_<table>_<column>` |
| Field di `relations[*].localKey` (belongsTo) | Index eksplisit dibuat dengan nama `idx_<table>_<column>` |

PostgreSQL dan Oracle tidak otomatis menghasilkan index untuk foreign key column, sehingga index eksplisit dari generator dibutuhkan. MySQL otomatis menghasilkan index untuk FK, namun generator tetap menulis DDL eksplisit untuk konsistensi lintas dialect.

## Aturan Skip Emit untuk Index Redundan

Bila user mendeklarasikan field di `indexes` array tetapi kolom tersebut sudah punya implicit index dari sumber lain, generator tidak akan menulis `CREATE INDEX` untuk menghindari duplikasi. Sumber implicit index:

| Sumber implicit | Penjelasan |
|-----------------|------------|
| Field dengan constraint `unique` (single column) | UNIQUE constraint sudah menghasilkan index di semua dialect |
| Field dengan constraint `pk` | PRIMARY KEY constraint sudah menghasilkan index di semua dialect |
| Field di `uniques` (composite) | Composite UNIQUE constraint sudah menghasilkan index di semua dialect |
| Field dengan `fk:` shorthand atau di `relations[*].localKey` | Auto-emit dari aturan FK auto-index di atas |

**Contoh:**

```javascript
fields: {
  po_number:   'string:50 unique notnull',     // unique → implicit index
  supplier_id: 'string:36 notnull',
  status:      "string:20 default:'draft'"
},
indexes: [
  'po_number',                                  // skip (sudah implicit dari unique)
  'supplier_id',                                // skip (sudah implicit dari FK relations)
  'status'                                      // emit
],
relations: {
  supplier: {
    type: 'belongsTo',
    localKey: 'supplier_id',
    references: 'supplier_id'
  }
}
```

Hasil: hanya 1 `CREATE INDEX` untuk `status` di output DDL.

Penulisan `index` eksplisit di field shorthand atau di array `indexes` tetap diperbolehkan dan tidak akan menyebabkan error (parser tidak melempar error). Generator akan melakukan deduplikasi berdasarkan kolom dan melewati yang redundan.

---

**Lihat juga**: [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
