# Field Foreign Key Constraints (`foreignKeyConstraints`)

Daftar foreign key tabel yang diturunkan dari relasi `belongsTo` di [SDF](../sdf/relations.md). Field ini hanya ditulis untuk Oracle dan SQLite, karena error foreign key kedua dialect itu tidak menyebut nama kolom.

## Ikhtisar

Saat `create` atau `update` mengirim nilai foreign key yang tidak ada di tabel induk, endpoint membalas `400 FOREIGN_KEY_VIOLATION` dengan `details` per kolom. Sumber nama kolomnya berbeda per dialect:

| Dialect | Sumber kolom di `details` |
|---------|---------------------------|
| PostgreSQL | Error driver database. Field ini tidak ditulis |
| MySQL | Error driver database. Field ini tidak ditulis |
| Oracle | Nama constraint di error `ORA-02291` dicocokkan ke field ini |
| SQLite | Endpoint memeriksa keberadaan data induk berdasarkan field ini sebelum menulis data |

Kontrak response ada di [Format Error Response 400 (Pelanggaran Constraint Database)](./field-validation.md#format-error-response-400-pelanggaran-constraint-database).

## Bentuk Data

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `name` | string | Nama constraint foreign key, sama dengan nama yang dibuat `schema migrate` |
| `columns` | array of string | Kolom foreign key di tabel ini (`localKey` relasi) |
| `refTable` | string | Tabel induk yang dirujuk |
| `refColumns` | array of string | Kolom tabel induk yang dirujuk (`references` relasi) |

SDF:

```javascript
relations: {
  category: { type: 'belongsTo', localKey: 'category_id', references: 'category_id' }
}
```

RDF hasil `payload generate` untuk Oracle atau SQLite:

```json
"foreignKeyConstraints": [
    { "name": "fk_guest_category", "columns": ["category_id"], "refTable": "category", "refColumns": ["category_id"] }
]
```

## Sumber

`payload generate` dan `payload sync` menurunkan field ini dari SDF di `--schema-path` (default folder `schema`). Field tidak perlu ditulis manual.

- Bila tabel tidak punya relasi `belongsTo`, field tidak ditulis.
- Bila SDF tidak ditemukan, field yang sudah ada dipertahankan apa adanya.

## Pengecekan Data Induk di SQLite

Endpoint SQLite memeriksa setiap kolom foreign key yang dikirim dengan nilai tidak kosong pada `create`, `update`, `create-composite`, dan `update-composite`. Pemeriksaan berjalan setelah validasi `fieldValidation` dan sebelum data ditulis. Kolom yang tidak dikirim pada `update` tidak diperiksa. Constraint foreign key di database tetap menjadi penjaga terakhir.

## Batasan

Response tetap `400 FOREIGN_KEY_VIOLATION`, tetapi tanpa `details`, pada kondisi berikut:

- Oracle: nama constraint di database berbeda dari nama di field ini, misalnya karena diubah langsung di database.
- Pelanggaran foreign key pada tabel detail endpoint composite.

---

**Lihat juga**: [`field-validation.md`](./field-validation.md) · [`check-constraints.md`](./check-constraints.md) · [`sdf/relations.md`](../sdf/relations.md) · [`commands/restforge-backend/payload/sync`](../../commands/restforge-backend/payload/sync.md) · [`rdf/`](./) · [`README`](../../README.md)
