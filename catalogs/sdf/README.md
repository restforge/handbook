# SDF — Schema Definition File

> Logical schema database yang dapat diterapkan ke struktur tabel.

## Ikhtisar

`dbschema-kit` adalah subproject di dalam ekosistem RESTForge yang bertanggung jawab atas dua hal:

1. Mendefinisikan struktur tabel database secara declarative dalam JavaScript
2. Memigrasikan struktur tersebut ke dialect target (PostgreSQL, MySQL, Oracle, SQLite) dengan tipe dan behavior native masing-masing dialect

Schema definition adalah deklarasi struktur tabel dalam format JavaScript object. Satu file schema mendeklarasikan satu model yang akan dipetakan ke satu tabel di database.



### Karakteristik Schema

| Aspek | Penjelasan |
|-------|-----------|
| Schema-as-code | Schema didefinisikan di file JavaScript |
| Inline shorthand | Type dan constraint diekspresikan dalam string ringkas yang dipisah spasi |
| Convention over configuration | Default value otomatis untuk field yang umum, hanya override yang berbeda |
| Dialect-agnostic vocabulary | Type bersifat logical bukan physical (misal `string:20`, bukan `varchar(20)`) |
| Native dialect behavior | DDL hasil migrasi memakai storage type native per dialect (misal boolean: PostgreSQL pakai `BOOLEAN`, MySQL pakai `VARCHAR(5)` menyimpan `'true'`/`'false'`) |
| Rujukan tunggal | Schema yang sama bisa menjadi sumber DDL migration dan payload RESTForge |

### Lingkup yang TIDAK Dicakup

- CRUD operation (ditangani oleh RESTForge generator)
- Query builder
- Object-relational mapping untuk runtime data access
- Lazy loading atau eager loading

## Struktur File Schema

Setiap model didefinisikan dalam satu file JavaScript di folder `schema/`. Nama file mengikuti nama tabel.

```
schema/
├── category.js
├── customer.js
├── supplier.js
├── warehouse.js
├── item-product.js
├── stock-inbound.js
├── stock-inbound-item.js
├── stock-outbound.js
├── stock-outbound-item.js
└── stock-beginning-balance.js
```

Setiap file mengekspor factory function yang menerima `{ defineModel }` dan mengembalikan hasil pemanggilan `defineModel()`:

```javascript
module.exports = ({ defineModel }) => defineModel('category', {
  fields: { /* ... */ },
  indexes: [ /* ... */ ]
});
```

**Mengapa factory function:** schema file tidak melakukan `require()` apa pun. CLI loader yang menyediakan `defineModel` sebagai parameter saat load. Pattern ini menghindari ketergantungan path npm package di file schema, sehingga schema file portable lintas environment (dev, CI, production) tanpa perlu install dbschema-kit sebagai dependency project.

## Anatomi `defineModel()`

```javascript
defineModel(tableName, options)
```

**Parameter:**

| Parameter | Tipe | Keterangan |
|-----------|------|-----------|
| `tableName` | string | Nama tabel di database, lowercase dengan underscore |
| `options` | object | Konfigurasi model (fields, indexes, relations, dll) |

**Properti `options`:**

| Properti | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `fields` | object | Ya | Definisi kolom dalam bentuk `{ nama_field: 'shorthand' }` |
| `schema` | string | Tidak | Schema namespace untuk multi-schema database (lihat halaman Schema Namespace) |
| `primaryKey` | array | Tidak | Composite primary key (lihat halaman terkait) |
| `indexes` | array | Tidak | Daftar field yang di-index |
| `uniques` | array | Tidak | Daftar composite unique constraint |
| `relations` | object | Tidak | Definisi foreign key relation |
| `checks` | array | Tidak | Daftar CHECK constraint |
| `softDelete` | object | Tidak | Aktivasi soft-delete beserta kolom reusable (lihat halaman Definisi Soft-Delete) |

## Navigasi

| Topik | File |
|-------|------|
| Sintaks Field Shorthand | [`field-shorthand.md`](./field-shorthand.md) |
| Sistem Tipe (Type System) | [`field-types.md`](./field-types.md) |
| Sistem Constraint | [`constraints.md`](./constraints.md) |
| Definisi Index | [`indexes.md`](./indexes.md) |
| Definisi Composite Unique | [`composite-unique.md`](./composite-unique.md) |
| Definisi CHECK Constraint | [`check-constraints.md`](./check-constraints.md) |
| Definisi Composite Primary Key | [`composite-primary-key.md`](./composite-primary-key.md) |
| Definisi Foreign Key Lanjutan | [`foreign-keys.md`](./foreign-keys.md) |
| Definisi Soft-Delete | [`soft-delete.md`](./soft-delete.md) |
| Kebijakan Inverse Relation | [`relations.md`](./relations.md) |
| Schema Namespace (Multi-Schema) | [`multi-schema.md`](./multi-schema.md) |
| Konvensi Penamaan | [`naming-convention.md`](./naming-convention.md) |
| Aturan Validasi | [`validation-rules.md`](./validation-rules.md) |
| Contoh Lengkap | [`complete-examples.md`](./complete-examples.md) |
| Workflow Sinkronisasi dengan Database | [`maintenance/sync-database.md`](./maintenance/sync-database.md) |

## Generator CLI

| Verb | Tujuan | Halaman Command |
|------|--------|-----------------|
| `restforge schema init` | Scaffold SDF baru | [`commands/restforge-backend/schema/init.md`](../../commands/restforge-backend/schema/init.md) |
| `restforge schema validate` | Validasi SDF | [`commands/restforge-backend/schema/validate.md`](../../commands/restforge-backend/schema/validate.md) |
| `restforge schema migrate` | Apply SDF ke database | [`commands/restforge-backend/schema/migrate.md`](../../commands/restforge-backend/schema/migrate.md) |
| `restforge schema introspect` | Reverse-engineer DB ke SDF | [`commands/restforge-backend/schema/introspect.md`](../../commands/restforge-backend/schema/introspect.md) |
| `restforge schema diff` | Drift detection | [`commands/restforge-backend/schema/diff.md`](../../commands/restforge-backend/schema/diff.md) |
| `restforge schema apply` | Apply drift incremental | [`commands/restforge-backend/schema/apply.md`](../../commands/restforge-backend/schema/apply.md) |

---

**Lihat juga**: [`catalogs/`](../) · [`README`](../../README.md)
