# Definisi Foreign Key Lanjutan (Advanced Foreign Key Definition)

Untuk konfigurasi foreign key yang lebih lengkap (ON DELETE, ON UPDATE behavior, atau target table dengan PK column non-konvensional), gunakan property `relations`:

```javascript
module.exports = ({ defineModel }) => defineModel('stock_inbound', {
  fields: {
    stock_inbound_id: 'string:36 pk',
    warehouse_id:     'string:36 notnull',
    supplier_id:      'string:36'
  },
  relations: {
    warehouse: {
      type: 'belongsTo',
      localKey: 'warehouse_id',
      references: 'warehouse_id',
      onDelete: 'restrict'
    },
    supplier: {
      type: 'belongsTo',
      localKey: 'supplier_id',
      references: 'supplier_id',
      onDelete: 'setNull'
    }
  }
});
```

Untuk kasus di mana satu tabel punya beberapa FK ke tabel yang sama (misal `created_by`, `updated_by`, `approved_by` semuanya merujuk ke tabel `user`), pakai property `target` untuk override target table:

```javascript
module.exports = ({ defineModel }) => defineModel('purchase_order', {
  fields: {
    purchase_order_id: 'string:36 pk',
    supplier_id:       'string:36 notnull',
    created_by:        'string:36 notnull',
    updated_by:        'string:36',
    approved_by:       'string:36'
  },
  relations: {
    supplier: {
      type: 'belongsTo',
      localKey: 'supplier_id',
      references: 'supplier_id'
    },
    creator: {
      type: 'belongsTo',
      target: 'user',
      localKey: 'created_by',
      references: 'id'
    },
    updater: {
      type: 'belongsTo',
      target: 'user',
      localKey: 'updated_by',
      references: 'id'
    },
    approver: {
      type: 'belongsTo',
      target: 'user',
      localKey: 'approved_by',
      references: 'id'
    }
  }
});
```

**Properti `relations[name]`:**

| Properti | Nilai Valid | Default | Keterangan |
|----------|-------------|---------|-----------|
| `type` | `belongsTo`, `hasMany`, `hasOne` | wajib | Tipe relasi |
| `target` | nama tabel target | nama key di `relations` object | Tabel yang dirujuk. Auto-derive dari nama relation bila tidak di-set |
| `localKey` | nama field di tabel ini | wajib | Field yang menyimpan FK di tabel current |
| `references` | nama field di tabel target | wajib | Field yang dirujuk di tabel target (umumnya PK) |
| `onDelete` | `cascade`, `restrict`, `setNull`, `noAction` | (tidak di-set) | Behavior saat parent dihapus. Bila tidak di-set, DDL skip clause `ON DELETE`, database pakai default-nya |
| `onUpdate` | `cascade`, `restrict`, `setNull`, `noAction` | (tidak di-set) | Behavior saat parent di-update. Bila tidak di-set, DDL skip clause `ON UPDATE`, database pakai default-nya |

Aturan derivasi `target`:

- Bila `target` di-set eksplisit, pakai nilai tersebut
- Bila tidak di-set, pakai nama key relation (misal `relations.supplier` → target `supplier`)
- Pattern multiple-FK-ke-target-yang-sama harus pakai `target` eksplisit karena nama key relation harus unik (misal `creator`, `updater`, `approver` semua ke `user`)

Catatan: `relations` di sini hanya untuk definisi FK constraint di DDL. Tidak ada implikasi runtime query (karena CRUD di-handle RESTForge generator).

## Mutual Exclusivity dengan `fk:` Shorthand

Untuk satu field, deklarasi FK hanya boleh dilakukan di **salah satu** tempat, tidak boleh di kedua tempat sekaligus:

| Skenario | Status | Behavior |
|----------|--------|----------|
| Field punya `fk:` shorthand saja | Valid | Auto-promote ke internal relations entry |
| Field di-reference oleh `relations[*].localKey` saja (tanpa `fk:` shorthand) | Valid | Pakai konfigurasi `relations` apa adanya |
| Field punya `fk:` shorthand DAN di-reference oleh `relations[*].localKey` | Error | Validator throw error eksplisit |
| Field tidak punya FK declaration sama sekali | Valid | Bukan FK, kolom biasa |

Pembagian penggunaan:

| Kasus | Cara Deklarasi |
|-------|----------------|
| FK tanpa custom `onDelete`/`onUpdate` (database pakai default) | `fk:` shorthand di field |
| FK dengan custom `onDelete` (cascade, set null, dst) | `relations` di options |
| FK dengan custom `onUpdate` | `relations` di options |
| Butuh nama relation custom untuk introspection downstream | `relations` di options |
| Relasi `hasOne` atau `hasMany` di parent | `relations` di options (tidak ada shorthand) |

## Auto-Promotion `fk:` Shorthand ke Internal Relations

Saat parser menemukan field dengan `fk:TABLE.COLUMN`, internal representation akan otomatis menambahkan entry di `relations` dengan default behavior. Aturan derivasi nama relation:

| Pattern field | Nama relation auto-generated | Catatan |
|---------------|------------------------------|---------|
| Field berakhiran `_id` (misal `category_id` → fk ke `category`) | Strip suffix `_id` (`category`) | Pattern paling umum |
| Field tidak berakhiran `_id` (misal `created_by` → fk ke `user`) | `<field>_<target_table>` (`created_by_user`) | Fallback agar tetap unik |

**Input developer:**

```javascript
module.exports = ({ defineModel }) => defineModel('item_product', {
  fields: {
    item_product_id: 'string:36 pk',
    category_id:     'string:36 fk:category.category_id notnull'
  }
});
```

**Internal representation setelah parse:**

```javascript
{
  fields: {
    item_product_id: { type: 'string', length: 36, pk: true, notnull: true },
    category_id:     { type: 'string', length: 36, notnull: true }
  },
  relations: {
    category: {
      type: 'belongsTo',
      target: 'category',
      localKey: 'category_id',
      references: 'category_id'
    }
  }
}
```

Auto-generated relation name yang collision dengan relation eksplisit akan di-throw oleh validator dengan pesan eksplisit (lihat halaman [`validation-rules.md`](./validation-rules.md)).

## Behavior pada `schema apply`

FK yang dideklarasikan di SDF di-resolve oleh `schema apply` mengikuti pola delta berikut:

| Skenario delta FK | Default behavior | Flag opt-in |
|-------------------|------------------|-------------|
| **Additive** (`onlyInSdf`) — FK ada di SDF, belum ada di DB | Emit `ALTER TABLE ... ADD CONSTRAINT ... FOREIGN KEY` | Tidak perlu (additive default) |
| **Drop** (`onlyInDb`) — FK ada di DB, hilang dari SDF | Skip dengan reason `requires --allow-drop` | `--allow-drop` |
| **Action change** (`mismatched`) — `onDelete`/`onUpdate` berbeda antara SDF dan DB | Skip dengan reason `requires --allow-modify`; saat di-apply: `DROP` lalu `ADD` constraint | `--allow-modify` |

**Naming constraint:**

- **ADD** (additive dan bagian ADD pada action change) memakai `generateConstraintName('fk', tableName, relName, maxIdentifierLength)`, **identik** dengan nama yang di-emit `CREATE TABLE` (full create / `schema migrate`). Konsekuensinya FK yang di-emit `schema apply` punya nama sama dengan FK dari full create untuk tabel/relasi yang sama.
- **DROP** (dan bagian DROP pada action change) memakai **nama constraint aktual dari introspeksi database** (`pg_constraint.conname` / `information_schema` / `user_constraints`), bukan nama hasil derivasi relName. Hal ini memastikan `DROP CONSTRAINT` menargetkan constraint yang benar-benar ada, termasuk FK yang dibuat via `fk:` shorthand (nama berbasis kolom) maupun via tool lain. Bila driver tidak menyediakan nama constraint, sistem fallback ke derivasi `generateConstraintName` (perilaku legacy).

**Order of operations:** `ADD FOREIGN KEY` terbit setelah `ADD COLUMN` dan `ADD CONSTRAINT UNIQUE`, sebelum `MODIFY COLUMN`. `DROP FOREIGN KEY` terbit sebelum `DROP CONSTRAINT UNIQUE` dan `DROP COLUMN` untuk menghindari dependency error. Pada action change, seluruh `DROP` FK terbit sebelum `ADD` FK pengganti.

**SQLite limitation:** SQLite tidak mendukung `ALTER TABLE ADD/DROP CONSTRAINT` untuk FK tanpa rebuild table. Semua perubahan FK di SQLite di-skip dengan reason `sqlite limitation` meskipun flag opt-in aktif; perubahan FK di SQLite harus diresolusi via re-create table.

---

**Lihat juga**: [`sdf/`](./) · [`maintenance/sync-database.md`](./maintenance/sync-database.md) · [`catalogs/`](../) · [`README`](../../README.md)
