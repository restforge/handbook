# RDF — Resource Definition File

> Kebutuhan fitur endpoint REST API yang akan di-generate.

## Ikhtisar

`@restforgejs/platform` menyediakan generator yang membaca RDF dan menghasilkan module backend siap pakai. RDF bertanggung jawab atas tiga hal:

1. Mendeklarasikan struktur endpoint REST API (tabel target, kolom yang diekspos, action yang aktif)
2. Mendeklarasikan aturan validasi per-field yang diterapkan pada operasi tulis (`create`, `update`)
3. Mendeklarasikan konfigurasi fitur tambahan (lookup, scope, master-detail, workflow, adjust, aggregate, kafka, component engine)

Satu file RDF dipetakan ke satu endpoint resource di runtime. Konvensi path fisik: `payload/<resource>.json` di root project.

> **Catatan istilah**: endpoint yang dihasilkan memakai pola action-based `POST /{resource}/{action}` (RPC-over-HTTP), bukan RESTful murni. Istilah "REST API" dipakai dalam pengertian luas industri. Penjelasan lengkap di [Catatan Arsitektur REST API](../../README.md#catatan-arsitektur-rest-api-rest-api-architecture-note).

RDF bekerja sebagai input untuk generator. Hasil generate adalah module JavaScript yang langsung dipasang oleh runtime server tanpa kompilasi ulang.

### Karakteristik RDF

| Aspek | Penjelasan |
|-------|-----------|
| Resource-as-config | Endpoint didefinisikan secara deklaratif di file JSON, bukan ditulis ulang per resource |
| Rujukan tunggal per resource | Satu file RDF menjadi referensi untuk seluruh endpoint resource (datatables, create, update, delete, dst) |
| Convention over configuration | Default value otomatis untuk action yang umum, hanya override yang berbeda |
| Dialect-agnostic | Generator menyesuaikan SQL per dialect target (PostgreSQL, MySQL, Oracle, SQLite) berdasarkan konfigurasi DB |
| Inline atau file reference | Query SQL dapat ditulis inline atau direferensikan via prefix `file:` |
| Validasi formal | RDF divalidasi di tahap generate dengan aturan terstruktur (lihat [`validation-rules.md`](./validation-rules.md)) |

### Lingkup yang TIDAK Dicakup

- Definisi struktur tabel database (ditangani oleh SDF di [`../sdf/`](../sdf/))
- Definisi halaman UI frontend (ditangani oleh UDF di [`../udf/`](../udf/))
- Custom business logic non-CRUD (ditangani oleh processor terpisah, lihat [`processor.md`](./processor.md))

## Struktur File RDF

Setiap resource dideklarasikan dalam satu file JSON di folder `payload/`. Nama file mengikuti nama endpoint (kebab-case direkomendasikan).

```
payload/
├── category.json
├── supplier.json
├── warehouse.json
├── item-product.json
├── stock-inbound.json
├── stock-outbound.json
└── query/
    ├── item-product-datatables.sql
    └── stock-inbound-detail.sql
```

Setiap file RDF adalah dokumen JSON valid dengan struktur object di level root:

```json
{
    "tableName": "category",
    "primaryKey": "category_id",
    "fieldName": ["category_id", "category_code", "category_name"],
    "action": {
        "datatables": true,
        "create": true,
        "update": true,
        "delete": true,
        "first": true,
        "lookup": true,
        "read": true
    }
}
```

Folder `payload/query/` adalah konvensi untuk menyimpan file SQL eksternal yang direferensikan via prefix `file:` dari field `datatablesQuery`, `viewQuery`, atau `exportQuery`.

## Anatomi RDF

Berikut daftar lengkap field top-level yang dikenali oleh generator. Field dikelompokkan berdasarkan tingkat kepentingan.

### Field Wajib

| Field | Tipe | Keterangan |
|-------|------|-----------|
| `tableName` | string | Nama tabel database. Mendukung format `schema.table` untuk dialect multi-schema |
| `fieldName` | array of string | Daftar kolom yang diekspos sebagai field API. Urutan menentukan prioritas tampilan |
| `action` | object | Konfigurasi endpoint yang akan di-generate (setiap key = flag boolean per action) |

### Field Opsional Umum

| Field | Tipe | Keterangan |
|-------|------|-----------|
| `primaryKey` | string | Nama kolom primary key. Default: `fieldName[0]` |
| `version` | string | Versi RDF untuk tracking perubahan |
| `description` | string | Deskripsi singkat tentang endpoint |
| `fieldValidation` | array of object | Aturan validasi per-field untuk operasi `create` dan `update` |
| `uniqueConstraints` | array of object | Pemetaan unique constraint (`{name, fields}`) hasil introspeksi database. Schema-derived, ditulis secara otomatis oleh `payload generate`/`sync`. Sumber field penyebab konflik pada response `409` |
| `fieldNameLookup` | object | Konfigurasi kolom `id` dan `text` untuk endpoint lookup |
| `defaultScope` | object | Filter otomatis untuk action `lookup` dan `read` |
| `auditColumns` | `false` / `null` / object | Override behavior kolom audit (`created_at`, `created_by`, `updated_at`, `updated_by`) |

### Field Sumber Data

| Field | Tipe | Keterangan |
|-------|------|-----------|
| `datatablesQuery` | string | SQL SELECT untuk endpoint `/datatables`. Inline atau `file:path/to.sql` |
| `datatablesWhere` | array of string | Kolom yang dapat dicari di endpoint `/datatables`. Tambahkan `"all"` untuk pencarian lintas-kolom |
| `viewQuery` | string | SQL SELECT alternatif untuk action read-only (`read`, `first`, `aggregate`, `lookup`) |
| `viewName` | string | Nama database VIEW. Prioritas lebih tinggi dari `viewQuery` |
| `exportQuery` | string | SQL SELECT khusus untuk endpoint `/export`. Default: fallback ke `SELECT {fieldName} FROM {tableName}` |
| `dateTimeFields` | object/array | Konfigurasi field bertipe datetime untuk parsing format input |
| `columnFormats` | object | Format tampilan kolom untuk endpoint `/export` (Excel) |

### Field Konfigurasi Fitur

| Field | Tipe | Keterangan |
|-------|------|-----------|
| `masterDetail` | object | Konfigurasi endpoint composite (header + detail) |
| `adjustConfig` | object | Konfigurasi endpoint `/adjust` (atomic increment/decrement) |
| `aggregateConfig` | object | Konfigurasi JOIN untuk endpoint `/aggregate` |
| `workflow` | object | Konfigurasi endpoint `/change-status` (state machine + hook) |
| `importConfig` | object | Konfigurasi endpoint `/import-upload`, `/import-preview`, `/import-commit` |
| `components` | array of object | Konfigurasi event lifecycle hooks (`onBeforeInsert`, `onAfterUpdate`, dll) |
| `kafka` | object | Konfigurasi Kafka event publishing pada operasi CRUD |
| `processor` | array of object | Definisi endpoint custom non-CRUD (alternatif untuk RDF tipe processor) |
| `softDelete` | object | Blok behavior soft-delete (`enabled`, `reusable`, `visibility`). Schema-derived dari SDF via `payload generate` |
| `softDeleteFkChecks` / `softDeleteFkParents` / `softDeleteFkChildren` / `softDeleteCascadeTree` | array / object | Metadata FK untuk semantik FK-aware soft-delete. Schema-derived, ditulis otomatis oleh `payload generate` |

## Navigasi

### Field Wajib

| Topik | File |
|-------|------|
| Field Wajib | [`mandatory-fields.md`](./mandatory-fields.md) |

### Field Validasi dan Data

| Topik | File |
|-------|------|
| Field Validasi (`fieldValidation`) | [`field-validation.md`](./field-validation.md) |
| Field Unique Constraints (`uniqueConstraints`) | [`unique-constraints.md`](./unique-constraints.md) |
| Field Lookup (`fieldNameLookup`) | [`field-lookup.md`](./field-lookup.md) |
| Field Default Scope (`defaultScope`) | [`default-scope.md`](./default-scope.md) |
| Field Audit Columns (`auditColumns`) | [`audit-columns.md`](./audit-columns.md) |
| Field Sumber Data | [`data-source.md`](./data-source.md) |

### Field Pola Khusus

| Topik | File |
|-------|------|
| Field Master Detail (`masterDetail`) | [`master-detail.md`](./master-detail.md) |
| Field Adjust (`adjustConfig`) | [`adjust-config.md`](./adjust-config.md) |
| Field Aggregate (`aggregateConfig`) | [`aggregate-config.md`](./aggregate-config.md) |
| Field Workflow (`workflow`) | [`workflow.md`](./workflow.md) |
| Field Soft-Delete (`softDelete`) | [`soft-delete.md`](./soft-delete.md) |

### Field Integrasi

| Topik | File |
|-------|------|
| Field Component Engine (`components`) | [`components.md`](./components.md) |
| Field Kafka (`kafka`) | [`kafka.md`](./kafka.md) |
| Field Import Config (`importConfig`) | [`import-config.md`](./import-config.md) |
| Format Referensi File (`file:` Prefix) | [`file-reference.md`](./file-reference.md) |
| Field Processor (`processor`) | [`processor.md`](./processor.md) |

### Konvensi dan Validasi

| Topik | File |
|-------|------|
| Konvensi Penamaan | [`naming-convention.md`](./naming-convention.md) |
| Aturan Validasi RDF | [`validation-rules.md`](./validation-rules.md) |
| Contoh Lengkap | [`complete-examples.md`](./complete-examples.md) |

## Generator CLI

| Verb | Tujuan | Halaman Command |
|------|--------|-----------------|
| `restforge payload generate` | Generate RDF dari struktur database | [`commands/restforge-backend/payload/generate.md`](../../commands/restforge-backend/payload/generate.md) |
| `restforge payload validate` | Validasi RDF terhadap struktur tabel | [`commands/restforge-backend/payload/validate.md`](../../commands/restforge-backend/payload/validate.md) |
| `restforge payload diff` | Diff kolom antara RDF dan database | [`commands/restforge-backend/payload/diff.md`](../../commands/restforge-backend/payload/diff.md) |
| `restforge payload sync` | Apply schema drift dari DB ke RDF | [`commands/restforge-backend/payload/sync.md`](../../commands/restforge-backend/payload/sync.md) |
| `restforge endpoint create` | Generate endpoint REST dari RDF | [`commands/restforge-backend/endpoint/create.md`](../../commands/restforge-backend/endpoint/create.md) |

---

**Lihat juga**: [`catalogs/`](../) · [`README`](../../README.md)
