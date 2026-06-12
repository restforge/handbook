# Aturan Validasi Schema

Saat schema dimuat, parser melakukan validasi berikut:

| Aturan | Pesan Error |
|--------|-------------|
| `tableName` wajib snake_case | `Table name must be snake_case` |
| `fields` tidak boleh kosong | `Model must have at least one field` |
| Tepat satu field dengan `pk` (jika tidak pakai composite) | `Model must have exactly one primary key field` |
| Tidak boleh ada `pk` shorthand bersamaan dengan property `primaryKey` | `Cannot use 'pk' shorthand together with 'primaryKey' option` |
| Tipe shorthand harus dikenali | `Unknown type: <type>` |
| Token constraint harus dikenali | `Unknown constraint '<token>' in field '<field>'` |
| `string` wajib punya length | `Type 'string' requires length modifier (e.g., string:20)` |
| `decimal` wajib punya precision dan scale | `Type 'decimal' requires precision and scale (e.g., decimal:15,2)` |
| FK target harus mengikuti format `table.column` | `Invalid foreign key format: <value>` |
| Field di `indexes` harus ada di `fields` | `Index references unknown field: <field>` |
| Field di `uniques` harus ada di `fields` | `Unique constraint references unknown field: <field>` |
| Field di `primaryKey` harus ada di `fields` | `Primary key references unknown field: <field>` |
| `relations[*].localKey` wajib dan harus ada di `fields` | `Relation '<name>' references unknown field: <field>` |
| `relations[*].references` wajib (string non-empty) | `Relation '<name>' missing 'references' property` |
| `relations[*].type` harus enum (`belongsTo`, `hasOne`, `hasMany`) | `Relation '<name>' has invalid type '<value>'` |
| `relations[*].onDelete` dan `relations[*].onUpdate` harus enum (`cascade`, `restrict`, `setNull`, `noAction`) | `Relation '<name>' has invalid <onDelete\|onUpdate> '<value>'` |
| Field tidak boleh punya `fk:` shorthand sekaligus di-reference oleh `relations[*].localKey` | `Field '<field>' has 'fk:' shorthand and is also referenced by relation '<relation_name>'. Use only one declaration method.` |
| Tidak boleh ada dua relations dengan `localKey` yang sama | `Field '<field>' is referenced by multiple relations: <name1>, <name2>` |
| Auto-generated relation name dari `fk:` shorthand tidak boleh collision dengan relation eksplisit | `Auto-generated relation name '<name>' from field '<field>' conflicts with explicit relation '<name>'. Rename one of them.` |
| `checks[*].field` wajib dan harus ada di `fields` | `Check at index <N> references unknown field: <field>` |
| `checks[*]` wajib punya tepat satu operator (`in`, `gt`, `gte`, `lt`, `lte`, `eq`, `neq`) | `Check at index <N> must have exactly one operator (got <count>)` |
| Operator `in` value harus array non-empty | `Check at index <N> operator 'in' requires non-empty array of values` |
| Operator comparison (`gt`, `gte`, `lt`, `lte`) value harus number | `Check at index <N> operator '<op>' requires numeric value` |
| Constraint `autoUpdate` hanya valid untuk type `timestamp` atau `date` (DEPRECATED rule, masih aktif untuk backward compatibility) | `Field '<field>': constraint 'autoUpdate' only valid for type 'timestamp' or 'date', got '<type>'` |
| Constraint `autoUpdate` tidak boleh dikombinasikan dengan `default:` (DEPRECATED rule, masih aktif untuk backward compatibility) | `Field '<field>': constraint 'autoUpdate' cannot be combined with 'default:'` |

> **Catatan**: Aturan validasi untuk constraint `autoUpdate` masih dipertahankan engine untuk backward compatibility. Constraint ini sudah DEPRECATED dan tidak boleh dipakai di template baru. Auto-update field `updated_at` ditangani di layer RDF runtime (BaseModel `auditColumns` helper), bukan di layer SDF. Detail lengkap dibahas di [`constraints.md`](./constraints.md#constraint-autoupdate-deprecated).

Validasi cross-model (misal FK target tabel ada, FK target column adalah PK di tabel target) dilakukan di tahap berbeda saat semua schema sudah dimuat.

## Aturan Validasi CLI

`restforge schema validate` menjalankan validasi tambahan di luar load-time. Berbeda dengan load-time validation yang melempar di error pertama, CLI mengumpulkan semua issue dalam satu run. Setiap issue memiliki `code` (machine-readable), `severity` (`error` atau `warning`), dan `message`.

Output JSON tersedia via `--format json` untuk konsumsi MCP, CI, atau LSP. Format kontrak stabil: `{ schemaVersion: '1.0', summary: { modelsLoaded, errorCount, warningCount }, issues: [...] }`.

### Cross-Model

| Code | Severity | Trigger |
|------|----------|---------|
| `relation-invalid-target` | error | `relations[*].target` kosong atau bukan string |
| `relation-unknown-target` | error | Target tabel tidak ditemukan di set schema yang dimuat |
| `relation-ambiguous-target` | error | Target unqualified cocok dengan lebih dari satu schema namespace |
| `relation-unknown-references-field` | error | `relations[*].references` tidak ada di tabel target |
| `relation-not-primary-key` | error | `belongsTo` references field yang bukan primary key di tabel target |

### Type Compatibility (FK)

| Code | Severity | Trigger |
|------|----------|---------|
| `fk-type-mismatch` | error | `localKey` source dan `references` target punya base type berbeda |
| `fk-length-mismatch` | warning | Sama-sama `string`/`text` tapi `length` berbeda (atau salah satu unspecified) |
| `fk-precision-mismatch` | warning | Sama-sama `decimal` tapi `precision` berbeda |
| `fk-scale-mismatch` | warning | Sama-sama `decimal` tapi `scale` berbeda |

### Circular Relation

| Code | Severity | Trigger |
|------|----------|---------|
| `circular-mandatory-relation` | error | Cycle dengan semua edge mandatory (insert impossible), atau self-reference dengan FK `notnull` |
| `circular-relation-nullable` | warning | Multi-table cycle dengan setidaknya satu nullable edge |

Self-reference nullable (misal `employee.manager_id` nullable) tidak ditandai karena pola tree/hierarchy umum dan dapat di-insert dengan FK NULL pada root row.

### Check Constraint Compatibility

| Code | Severity | Trigger |
|------|----------|---------|
| `check-op-incompatible-with-type` | error | Operator comparison (`gt`, `gte`, `lt`, `lte`) digunakan pada field non-numeric / non-date (`boolean`, `string`, `text`, `uuid`, `json`) |
| `check-value-type-mismatch` | error | Tipe JavaScript value tidak sesuai dengan field type. Untuk `in`, posisi item invalid disebut spesifik di message |

### Naming Convention (Opt-in via `--dialect`)

| Code | Severity | Trigger |
|------|----------|---------|
| `reserved-keyword-table` | warning | Nama tabel (case-insensitive) cocok dengan reserved keyword dialect target |
| `reserved-keyword-field` | warning | Nama field (case-insensitive) cocok dengan reserved keyword dialect target |
| `identifier-too-long` | warning | Nama tabel atau field melebihi `maxIdentifierLength` dialect target |

Naming rule hanya aktif dengan flag `--dialect <postgres|mysql|oracle|sqlite>`. Limit identifier diambil dari dialect modules di `lib/dbschema-kit/dialect/` sehingga warning muncul pada batas yang sama dengan emitter DDL.

### Schema Load Error

| Code | Severity | Trigger |
|------|----------|---------|
| `schema-load-error` | error | Path schema tidak ditemukan, file invalid, atau factory function melempar saat di-evaluasi |

### Severity dan Exit Code

| Kondisi | Default exit | Dengan `--strict-warnings` |
|---------|--------------|-----------------------------|
| 0 issue | 0 | 0 |
| Hanya warning | 0 | 1 |
| Ada error | 1 | 1 |
| Usage error (flag invalid) | 2 | 2 |

Warning disembunyikan secara default. Tambahkan `--strict` agar warning ditampilkan di output (human dan JSON). `--strict-warnings` otomatis mengaktifkan `--strict` dan mengangkat warning menjadi exit code 1.

---

**Lihat juga**: [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
