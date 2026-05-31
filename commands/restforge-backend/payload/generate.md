# `payload generate`

> Generate file payload JSON berdasarkan introspeksi tabel database.

## Pattern

```
npx restforge payload generate --config=<FILE> --table=<NAME> [options]
```

## Flag Wajib (Required)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--config <FILE>` | - | File config database (`.env`) |
| `--table <NAME>` | - | Nama tabel spesifik yang akan di-generate payload-nya |

## Flag Opsional (Optional)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--output <DIR>` | `payload/` | Output directory untuk file payload yang di-generate |

## Contoh (Examples)

```bat
:: Generate untuk satu tabel ke folder default
npx restforge payload generate --config=db.env --table=users

:: Generate untuk satu tabel ke folder custom
npx restforge payload generate --config=db.env --table=users --output=./my-payloads
```

## Field Validation Otomatis

Command `payload generate` melakukan introspeksi schema database lalu men-derive `fieldValidation` untuk setiap field di output payload. Tujuannya menyediakan baseline validasi yang konsisten dengan constraint database sehingga endpoint `/create` dan `/update` menolak input invalid di application layer sebelum SQL dieksekusi.

### Sumber Introspeksi (Introspection Sources)

| Method | Data yang Diambil |
|---|---|
| `getDetailedColumnInfo()` | `data_type`, `udt_name`, `column_default`, `is_nullable`, `character_maximum_length`, `numeric_precision`, `numeric_scale` |
| `getConstraints()` | Daftar PRIMARY KEY dan UNIQUE constraint beserta kolom-kolomnya |

### Constraint Per Tipe Database

| Tipe Database | Output `type` | Derived Constraints |
|---|---|---|
| `UUID` native (PostgreSQL `uuid`) sebagai PK | `uuid` | `primaryKey: true`, `autoGenerate: true` |
| `VARCHAR`/`TEXT` sebagai PK (tanpa `nextval`) | `string` | `primaryKey: true`, `autoGenerate: true`, `required: true`, `maxLength: <N>` jika `character_maximum_length` tersedia |
| `INTEGER`, `BIGINT`, `SMALLINT`, `SERIAL` (non-PK) | `integer` | `default` jika ada DEFAULT literal |
| `NUMERIC`, `DECIMAL`, `REAL`, `DOUBLE PRECISION` | `number` | `precision` jika `numeric_scale > 0`, `default` jika ada |
| `BOOLEAN` | `boolean` | `default: true/false` jika ada |
| `DATE` | `date` | `format: 'dd/MM/yyyy'` |
| `TIMESTAMP`, `TIMESTAMPTZ`, `DATETIME` | `datetime` | `format: 'dd/MM/yyyy HH:mm:ss'`, `autoGenerate: true` jika DEFAULT `now()`/`CURRENT_TIMESTAMP`/`SYSTIMESTAMP` |
| `TIME` | `time` | `format: 'HH:mm:ss'` |
| `VARCHAR`/`TEXT`/`CHAR` (non-PK) | `string` | `required: true` jika `NOT NULL`, `maxLength: <N>` dari `character_maximum_length`, `unique: true` jika single-column UNIQUE constraint |

### Behavior Khusus String Field

Field string non-PK hanya mendapat entry di `fieldValidation` bila memiliki minimal satu constraint actionable. Nullable string tanpa `character_maximum_length` dan tanpa UNIQUE constraint di-skip dari output agar payload tidak bloat dengan entry yang tidak menambah validasi runtime.

Constraint UNIQUE composite (multi-column, contoh `UNIQUE (email, tenant_id)`) tidak di-derive karena uniqueness berlaku pada kombinasi kolom, bukan kolom individual. Constraint semacam ini tetap di-enforce oleh database, namun tidak tercatat di `fieldValidation`.

MySQL `auto_increment` primary key dan PostgreSQL `serial` primary key (yang menggunakan `nextval`) di-skip dari `fieldValidation` karena PK di-handle oleh database, bukan application layer.

### Customisasi Manual (Manual Customization)

Output `payload generate` adalah baseline. Constraint tambahan seperti `format`, `pattern`, `min`/`max`, `enum`, atau preset `email`/`phone`/`url`/`uuid` perlu ditambahkan secara manual di file payload setelah generate. Detail constraint per tipe ada di [`catalogs/rdf/field-validation`](../../../catalogs/rdf/field-validation.md).

---

**Lihat juga**: [`payload/`](./) · [`commands/`](../) · [`README`](../../../README.md)
