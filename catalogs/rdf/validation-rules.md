# Aturan Validasi RDF

Saat generator membaca RDF, validasi berikut diterapkan:

## Validasi Struktur Top-Level

| Aturan | Pesan Error |
|--------|-------------|
| `tableName` wajib (string non-empty) | `tableName is required` |
| `fieldName` wajib (array dengan minimal 1 item) | `fieldName must be a non-empty array` |
| `action` wajib (object) | `action is required and must be an object` |
| `action` hanya boleh berisi key yang dikenali | `Unknown action: <key>` |
| `primaryKey` (jika ditetapkan) harus ada di `fieldName` | `primaryKey '<col>' is not in fieldName` |

## Validasi `fieldValidation`

| Aturan | Pesan Error |
|--------|-------------|
| `name` wajib dan harus ada di `fieldName` | `fieldValidation.name '<col>' is not in fieldName` |
| `type` wajib dan harus enum valid | `fieldValidation.type '<type>' is not supported` |
| Constraint per tipe harus sesuai (misal `minLength` hanya untuk `string`) | `Constraint '<name>' is not applicable to type '<type>'` |

## Validasi `defaultScope`

| Aturan | Pesan Error |
|--------|-------------|
| Harus object | `defaultScope must be an object` |
| Key harus `lookup` atau `read` (key lain = warning) | `Warning: defaultScope key '<key>' is not recognized` |
| Kolom harus ada di `fieldName` | `defaultScope.<action> references column '<col>' which is not in fieldName` |
| Nilai harus boolean/string/number | `defaultScope.<action>.<col> must be a boolean, string, or number` |

## Validasi `auditColumns`

| Aturan | Pesan Error |
|--------|-------------|
| Nilai harus `false`, `null`, atau object (bukan array) | `Property 'auditColumns' in <file> must be false, null, or object` |
| Object value harus mengikuti shape `{ createdAt, createdBy, updatedAt, updatedBy }` | `Invalid auditColumns value for <table>: must be false, null, or object` (emit oleh template generator) |

## Validasi `action` Dependency

| Aturan | Pesan Error |
|--------|-------------|
| `createComposite`/`updateComposite`/`readComposite` aktif tanpa `masterDetail` | `Composite action requires masterDetail configuration` |
| `adjust` aktif tanpa `adjustConfig` | `Action 'adjust' requires adjustConfig` |
| `aggregate` aktif tanpa `aggregateConfig.joins` | (Warning) `aggregate action active without joins defined` |
| `workflow` aktif tanpa kolom status di `fieldName` | `workflow.statusField '<col>' is not in fieldName` |
| `import` aktif tanpa `importConfig` | (Warning) `import action active without importConfig` |

## Validasi File Reference

| Aturan | Pesan Error |
|--------|-------------|
| File reference (`file:...`) harus ada di filesystem | `Referenced file not found: payload/<path>` |
| File harus dapat dibaca | `Cannot read referenced file: <path>` |

## Validasi Schema Database

Selain validasi shape RDF di atas, command `restforge endpoint create`
melakukan cross-check antara payload dan struktur tabel
database aktual sebelum codegen dimulai. Mode validasi ini *audit-column-aware*
dan *query-source-aware*:

| Aturan | Pesan Error |
|--------|-------------|
| Setiap kolom di `fieldName` harus ada di kolom valid (lihat definisi di bawah) | `[-] <col> (in payload, not in database)` |
| Kolom audit (`created_at`, `created_by`, `updated_at`, `updated_by`) harus ada di database bila `auditColumns` aktif | `[+] <cols> (required by auditColumns=true, not in database)` |
| Kolom database non-audit yang tidak ada di payload dilaporkan | `[+] <col> (in database, not in payload)` |
| Tabel target harus ada di database | `Table "<table>" not found in database` |
| Tipe kolom payload harus cocok dengan tipe kolom database (jika `fieldValidation` ditetapkan) | `[~] <col> (type: <payloadType> -> <dbType>)` |

### Definisi Kolom Valid

Validator menghitung kolom valid sebagai **UNION** dari
beberapa sumber, bukan hanya kolom fisik tabel utama. Hal ini sesuai dengan
spec [data-source.md](./data-source.md) yang mengizinkan `fieldName`
berisi kolom hasil JOIN dari `viewQuery`/`datatablesQuery`/`exportQuery`
(misal `category_name` di endpoint `item_product`).

Kolom valid = UNION dari sumber berikut:

| Sumber | Mekanisme Resolusi | Kapan Dipakai |
|--------|-------------------|---------------|
| Kolom fisik `tableName` | `information_schema.columns` introspection | Selalu (dasar drift check) |
| Kolom dari `viewName` | Introspection terhadap database VIEW | Jika `viewName` ditetapkan |
| Kolom output `viewQuery` | Describe query via wrapper `SELECT * FROM (<sql>) WHERE false` | Jika `viewQuery` ditetapkan |
| Kolom output `datatablesQuery` | Sama dengan `viewQuery` | Jika `datatablesQuery` ditetapkan |
| Kolom output `exportQuery` | Sama dengan `viewQuery` | Jika `exportQuery` ditetapkan |

Mekanisme wrapper memanfaatkan database engine untuk resolve metadata kolom
output (termasuk alias dan JOIN) tanpa konsumsi row. Tidak ada SQL parser
manual, sehingga edge case seperti CTE, subquery, dan fungsi tertangani
sebagaimana database engine menangani query yang sama.

**Catatan tentang `detailQuery`:** Properti `masterDetail.detailConfig.detailQuery`
TIDAK ikut di-resolve di scope root. Master-detail punya `fieldName` terpisah di
`masterDetail.detailConfig.fieldName` yang divalidasi di scope detail sendiri.

### Toleransi terhadap Query Source Error

Jika query source gagal di-describe (SQL syntax error, file tidak ditemukan,
view tidak ada, dll), validator mencatat sebagai warning tetapi tetap melanjutkan
drift check dengan UNION dari sumber yang berhasil. Output drift report
menyertakan bagian warning:

```
  Query source warnings (column resolution may be incomplete):
    [!] viewQuery: Referenced SQL file not found: query/item-product-view.sql
    [!] exportQuery: 42703: column "kategori" does not exist
```

Trade-off: false-positive drift mungkin terjadi jika query rusak, namun user
mendapat warning eksplisit untuk memperbaiki query secara terpisah. Untuk bypass
penuh, gunakan flag `--skip-schema-check`.

### Pesan Sukses Validasi

Saat seluruh validasi lulus, output menampilkan jumlah kolom valid
(termasuk dari query sources):

```
Schema Validation:
  Status       OK (22 columns match database)
```

Angka kolom di atas adalah UNION dari kolom fisik + kolom dari seluruh
query sources. Jadi 22 kolom pada contoh `item_product` mencerminkan
kolom fisik tabel `item_product` ditambah kolom hasil JOIN dari `viewQuery`
(termasuk `category_name`).

Behavior saat drift terdeteksi:

- Command exit dengan kode `1`
- Tidak ada file output yang ditulis (no partial generate)
- Tidak ada archive yang dibuat (`*.archive.NNN`)

Bypass validation (escape hatch untuk DB offline / maintenance):

```bat
npx restforge endpoint create ... --skip-schema-check
```

Flag `--skip-schema-check` melakukan bypass validasi schema secara universal,
tidak peduli apakah `--config` disediakan atau tidak. Validator melakukan early-return
dengan status `skipped` SEBELUM mencoba memuat config file atau menyambung ke
database. Skenario yang valid:

- `--skip-schema-check` tanpa `--config` → skip schema check
- `--skip-schema-check` dengan `--config=foo.env` → skip schema check (config diabaikan)

Validasi tambahan tersedia via command `restforge payload validate` dan
`restforge payload diff` (lihat [`commands/restforge-backend/payload/diff.md`](../../commands/restforge-backend/payload/diff.md)).

### Resolver Kanonik untuk Drift Audit Columns

`restforge payload sync` adalah **resolver kanonik** untuk drift kolom audit.
Sync mendeteksi ketidakselarasan antara `auditColumns` di
payload dan keberadaan kolom audit di database, lalu secara otomatis menetapkan
`"auditColumns": false` bila tabel tidak punya kolom audit standar. Detail
matriks resolusi lihat
[`commands/restforge-backend/payload/sync.md`](../../commands/restforge-backend/payload/sync.md#resolusi-audit-columns-audit-columns-resolution).

Setelah sync, `endpoint create` lulus validasi schema tanpa user perlu mengedit
file payload secara manual.

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
