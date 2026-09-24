# Field Unique Constraints (`uniqueConstraints`)

Daftar unique constraint tabel hasil introspeksi database, dalam bentuk pemetaan
nama constraint ke kolom pembentuknya. Field bersifat schema-derived (turunan struktur
database), sejajar dengan [`fieldValidation`](./field-validation.md), dan ditulis secara otomatis
oleh generator. Field ini menjadi sumber pemetaan yang memungkinkan response `409`
duplicate entry menyebutkan field penyebab konflik pada dialect MySQL dan Oracle.

## Ikhtisar

Saat operasi tulis (`create`, `update`, `adjust`, dan varian composite) melanggar unique
constraint, handler endpoint mengembalikan HTTP status `409`. Agar response dapat menyebut
field penyebab konflik secara seragam di keempat dialect, runtime perlu memetakan nama
constraint yang dilaporkan driver database ke nama field logis.

Pada PostgreSQL dan SQLite, daftar kolom tersedia langsung pada error object driver,
sehingga handler dapat membacanya saat runtime. Pada MySQL dan Oracle, driver hanya
melaporkan nama constraint (bukan daftar kolom), sehingga handler memerlukan pemetaan
`nama constraint → kolom` yang disiapkan saat generate. Field `uniqueConstraints`
menyimpan pemetaan tersebut di RDF.

Kontrak response `409` lengkap dijelaskan di
[Format Error Response 409 (Duplicate Entry)](./field-validation.md#format-error-response-409-duplicate-entry).

## Bentuk Data

`uniqueConstraints` adalah array of object. Setiap entry mewakili satu unique constraint.

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `name` | string | Nama constraint UNIQUE aktual di database (hasil introspeksi) |
| `fields` | array of string | Daftar kolom pembentuk constraint. Single UNIQUE berisi satu kolom, composite UNIQUE berisi dua kolom atau lebih |

```json
{
    "uniqueConstraints": [
        { "name": "uq_visitors_email", "fields": ["email"] }
    ]
}
```

## Contoh

### Single UNIQUE

Constraint pada satu kolom:

```json
"uniqueConstraints": [
  { "name": "uq_visitors_email", "fields": ["email"] }
]
```

### Composite UNIQUE

Constraint pada kombinasi beberapa kolom:

```json
"uniqueConstraints": [
  { "name": "uq_orders_wh_sku", "fields": ["warehouse_id", "sku"] }
]
```

### Kosong atau Absen

Bila tabel tidak memiliki unique constraint, field ditulis sebagai array kosong:

```json
"uniqueConstraints": []
```

Bila generate dijalankan tanpa akses database (`--skip-schema-check`), field tidak ditulis
sama sekali (absen dari RDF). Baik bentuk kosong (`[]`) maupun absen diperlakukan identik
oleh konsumen: handler MySQL/Oracle beralih ke fallback dan menghilangkan `details` pada response
`409` (lihat [Batasan](#batasan)).

## Sumber

Field ditulis secara otomatis oleh perintah `payload generate` dan `payload sync` dari hasil
introspeksi database. Introspector membaca nama dan kolom constraint UNIQUE aktual di
database (single dan composite), lalu generator menanamnya ke RDF apa adanya tanpa
menurunkan ulang nama dari konvensi penamaan.

| Perintah | Halaman Command |
|----------|-----------------|
| `restforge payload generate` | [`commands/restforge-backend/payload/generate.md`](../../commands/restforge-backend/payload/generate.md) |
| `restforge payload sync` | [`commands/restforge-backend/payload/sync.md`](../../commands/restforge-backend/payload/sync.md) |

Field **tidak perlu ditulis secara manual**. Saat struktur unique constraint di database berubah,
regenerasi field dilakukan via `payload sync`.

## Konsumen

| Dialect | Pemakaian `uniqueConstraints` |
|---------|-------------------------------|
| MySQL | Menanam pemetaan `nama → fields` ke handler endpoint. Driver melaporkan nama constraint, sehingga pemetaan diperlukan untuk menemukan field penyebab konflik |
| Oracle | Sama dengan MySQL |
| PostgreSQL | Tidak memakai field. Handler membaca daftar kolom langsung dari error driver saat runtime |
| SQLite | Tidak memakai field. Sama dengan PostgreSQL |

Pada PostgreSQL dan SQLite, field tetap ditulis ke RDF demi keseragaman struktur, namun
dapat dihilangkan dari RDF tanpa efek pada response kedua dialect tersebut.

## Interaksi dengan Drift Detection

Field `uniqueConstraints` tidak ikut dibandingkan oleh `payload validate` maupun
`payload diff`, sehingga keberadaan atau perubahannya tidak memicu drift. Pembanding drift
hanya menyentuh daftar kolom (`fieldName`), kolom audit, dan `fieldValidation`. Perubahan
unique constraint di database tercermin di RDF saat `payload sync` berikutnya dijalankan.

## Batasan

Beberapa skenario tabel tidak tercakup oleh pemetaan ini. Pada skenario tersebut response
tetap valid `409` dengan `details` dihilangkan (fallback aman, tidak ada regresi):

- Unique diterapkan via `CREATE UNIQUE INDEX` (bukan CONSTRAINT) yang tidak muncul di
  `information_schema.table_constraints`.
- Partial atau expression-based unique index, misal `CREATE UNIQUE INDEX ON t (lower(email))`.
- Constraint diganti namanya di database setelah RDF di-generate, tanpa `payload sync` ulang.
- Constraint baru ditambahkan di database tanpa `payload sync`.

Pada keempat skenario, handler tetap mengembalikan status `409` dengan `error` dan `message`
statis, hanya tanpa map `details`.

---

**Lihat juga**: [`field-validation.md`](./field-validation.md) · [`commands/restforge-backend/payload/generate`](../../commands/restforge-backend/payload/generate.md) · [`commands/restforge-backend/payload/sync`](../../commands/restforge-backend/payload/sync.md) · [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
