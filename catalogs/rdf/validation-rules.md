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

## Aturan `datatablesWhere`

Setiap entri `datatablesWhere` diperiksa saat generate, karena daftar itu adalah whitelist `searchBy` di runtime. Sebelumnya, entri yang tidak pernah cocok dengan kolom yang tersedia dibuang diam-diam oleh runtime, sehingga pencarian pada kolom itu tidak pernah bekerja dan tidak ada pesan apa pun yang menjelaskan kenapa.

Kolom pembanding ditentukan lebih dulu:

| Kondisi `datatablesQuery` | Kolom pembanding |
|---------------------------|------------------|
| Inline SQL yang seluruh segmen SELECT-nya berupa kolom sederhana (`col`, `alias.col`, opsional `AS alias_output`) | Irisan `fieldName` dengan nama kolom hasil SELECT |
| Tidak ditetapkan, berupa referensi `file:`, atau memuat ekspresi kompleks sehingga tidak dapat diurai andal | Seluruh `fieldName` |

Bila soft-delete aktif, kolom soft-delete ikut dianggap sah karena runtime menambahkannya ke seluruh proyeksi.

| Aturan | Pesan Error |
|--------|-------------|
| Setiap entri harus string non-empty | `Property 'datatablesWhere' in <file> must contain only non-empty strings, found: <nilai>` |
| Entri `all` selalu sah, tidak dicocokkan ke kolom mana pun | — |
| Entri tidak boleh memakai prefiks alias tabel, karena runtime mencocokkan nama kolom hasil SELECT | `datatablesWhere entry '<entri>' in <file> uses a table alias prefix. The runtime matches search columns against the SELECT result column names, so aliased entries are silently dropped. Use the SELECT result column name instead: '<kolom>'` |
| Entri beralias yang tetap tidak cocok setelah prefiksnya dibuang | `datatablesWhere entry '<entri>' in <file> uses a table alias prefix and does not match any valid column. Valid columns: [<daftar>] (or 'all')` |
| Entri lain harus cocok dengan kolom pembanding, tanpa memperhatikan besar-kecil huruf | `datatablesWhere entry '<entri>' in <file> does not match <sumber>. The runtime would silently drop it and searching on it would never work. Valid columns: [<daftar>] (or 'all')` |

Bagian `<sumber>` pada pesan terakhir terisi `the SELECT result columns of datatablesQuery` atau `fieldName`, mengikuti kolom pembanding yang dipakai.

> **Peringatan upgrade:** Aturan ini memutus payload lama. Payload lapangan yang menulis `datatablesWhere` dengan prefiks alias (misalnya `a.supplier_code`) sebelumnya tetap lolos generate walau entri itu tidak pernah berfungsi, sekarang menghentikan `endpoint create` dan `payload validate`. Perbaikannya adalah menyunting entri menjadi nama kolom hasil SELECT yang disebutkan pesan error, misalnya `a.supplier_code` menjadi `supplier_code`. Jangan mengandalkan `payload sync` untuk pembersihan ini karena sync mencocokkan `datatablesWhere` hanya terhadap kolom fisik tabel bertipe string, sehingga kolom hasil JOIN yang sah ikut terbuang. Konsekuensi operasionalnya dijelaskan di [`endpoint create`](../../commands/restforge-backend/endpoint/create.md#validasi-schema-database) dan [`payload validate`](../../commands/restforge-backend/payload/validate.md#logika-validasi).

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

| Sumber | Cara Kolom Dibaca | Kapan Dipakai |
|--------|-------------------|---------------|
| Kolom fisik `tableName` | Struktur tabel di database | Selalu (dasar drift check) |
| Kolom dari `viewName` | Struktur VIEW di database | Jika `viewName` ditetapkan |
| Kolom output `viewQuery` | Kolom hasil query, dibaca tanpa mengambil baris data | Jika `viewQuery` ditetapkan |
| Kolom output `datatablesQuery` | Sama dengan `viewQuery` | Jika `datatablesQuery` ditetapkan |
| Kolom output `exportQuery` | Sama dengan `viewQuery` | Jika `exportQuery` ditetapkan |

Kolom hasil query dibaca langsung dari database, sehingga alias, JOIN, CTE,
subquery, dan fungsi dikenali sama seperti saat query dijalankan.

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
- Tidak ada file yang diarsipkan ke `.restforge/archive/`

Bypass validation (escape hatch untuk DB offline / maintenance):

```bat
npx restforge endpoint create ... --skip-schema-check
```

Flag `--skip-schema-check` melakukan bypass validasi schema secara universal,
tidak peduli apakah `--config` disediakan atau tidak. Validasi schema berstatus `skipped`,
dan RESTForge tidak memuat config file maupun menyambung ke database. Skenario yang valid:

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
