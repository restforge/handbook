# Field Check Constraints (`checkConstraints`)

Daftar CHECK constraint tabel yang diturunkan dari [`checks` SDF](../sdf/check-constraints.md), dalam bentuk pemetaan nama constraint ke field, operator, dan nilainya. Field ditulis otomatis oleh generator, sejajar dengan [`uniqueConstraints`](./unique-constraints.md).

## Ikhtisar

Aturan CHECK di SDF juga diturunkan ke [`fieldValidation`](./field-validation.md#turunan-check-sdf), sehingga nilai yang melanggarnya biasanya sudah ditolak sebelum query dijalankan. Penolakan oleh database tetap bisa terjadi, misalnya bila batas di `fieldValidation` dihapus atau RDF belum di-sync. Pada kondisi itu, driver database hanya melaporkan nama constraint. Handler memakai `checkConstraints` untuk menemukan field penyebabnya dan menyusun pesan per field pada response `400`.

Kontrak response lengkap ada di [Format Error Response 400 (Pelanggaran Constraint Database)](./field-validation.md#format-error-response-400-pelanggaran-constraint-database).

## Bentuk Data

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `name` | string | Nama constraint CHECK, sama dengan nama yang dibuat `schema migrate` untuk dialect aktif |
| `field` | string | Field yang dibatasi |
| `op` | string | Operator SDF: `in`, `gt`, `gte`, `lt`, `lte`, `eq`, atau `neq` |
| `value` | any | Nilai pembanding. Array untuk `in`, nilai tunggal untuk operator lain |

```json
"checkConstraints": [
    { "name": "chk_guest_status", "field": "status", "op": "in", "value": ["waiting", "called", "done"] },
    { "name": "chk_guest_deposit_value", "field": "deposit_value", "op": "gt", "value": 0 },
    { "name": "chk_guest_channel", "field": "channel", "op": "neq", "value": "none" }
]
```

## Sumber

`payload generate` dan `payload sync` menurunkan field ini dari SDF di `--schema-path` (default folder `schema`). Field tidak perlu ditulis manual.

- Bila tabel di SDF tidak punya `checks`, field tidak ditulis.
- Bila SDF tidak ditemukan, field yang sudah ada dipertahankan apa adanya.

## Pesan per Operator

| Operator | Pesan di `details` |
|----------|--------------------|
| `gt` | `Field '<field>' must be greater than <value>` |
| `gte` | `Field '<field>' must be at least <value>` |
| `lt` | `Field '<field>' must be less than <value>` |
| `lte` | `Field '<field>' must not exceed <value>` |
| `eq` | `Field '<field>' must be equal to <value>` |
| `neq` | `Field '<field>' must not be equal to <value>` |
| `in` | `Field '<field>' must be one of: <value1>, <value2>` |

## Batasan

Response tetap `400`, tetapi tanpa `details`, pada kondisi berikut:

- CHECK dibuat langsung di database tanpa dideklarasikan di SDF.
- Nama constraint diubah di database, atau `schema migrate` dijalankan dengan `--max-name-length` sehingga nama berbeda dari hasil penamaan default.

---

**Lihat juga**: [`field-validation.md`](./field-validation.md) · [`foreign-key-constraints.md`](./foreign-key-constraints.md) · [`sdf/check-constraints.md`](../sdf/check-constraints.md) · [`commands/restforge-backend/payload/sync`](../../commands/restforge-backend/payload/sync.md) · [`rdf/`](./) · [`README`](../../README.md)
