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
| `--schema-path <PATH>` | `schema` | Lokasi SDF (file atau folder). Wajib memuat deklarasi tabel bila tabel memiliki kolom soft-delete, karena blok `softDelete` RDF diturunkan dari SDF. Diabaikan untuk tabel non-soft-delete |

## Contoh (Examples)

```bat
:: Generate untuk satu tabel ke folder default
npx restforge payload generate --config=db.env --table=users

:: Generate untuk satu tabel ke folder custom
npx restforge payload generate --config=db.env --table=users --output=./my-payloads

:: Generate tabel soft-delete dengan lokasi SDF eksplisit
npx restforge payload generate --config=db.env --table=category --schema-path=./schema
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

Field string non-PK hanya mendapat entry di `fieldValidation` bila memiliki minimal satu constraint actionable. Nullable string tanpa `character_maximum_length` dan tanpa UNIQUE constraint dilewati dari output agar payload tidak mengalami pembengkakan dengan entry yang tidak menambah validasi runtime.

Constraint UNIQUE composite (multi-column, contoh `UNIQUE (email, tenant_id)`) tidak diturunkan karena uniqueness berlaku pada kombinasi kolom, bukan kolom individual. Constraint semacam ini tetap di-enforce oleh database, namun tidak tercatat di `fieldValidation`.

MySQL `auto_increment` primary key dan PostgreSQL `serial` primary key (yang menggunakan `nextval`) dilewati dari `fieldValidation` karena PK ditangani oleh database, bukan application layer.

### Customisasi Manual (Manual Customization)

Output `payload generate` adalah baseline. Constraint tambahan seperti `format`, `pattern`, `min`/`max`, `enum`, atau preset `email`/`phone`/`url`/`uuid` perlu ditambahkan secara manual di file payload setelah generate. Detail constraint per tipe ada di [`catalogs/rdf/field-validation`](../../../catalogs/rdf/field-validation.md).

## Default Scope Otomatis (`is_active`)

Bila tabel memiliki kolom `is_active`, `payload generate` otomatis menambahkan `defaultScope` built-in ke output payload:

```json
"defaultScope": {
    "lookup": { "is_active": true },
    "read": { "is_active": true }
}
```

Akibatnya, endpoint `lookup` dan `read` otomatis hanya menampilkan record aktif tanpa perlu menulis filter secara manual. Action `datatables` dan `first` tidak terpengaruh. Bila tabel tidak memiliki kolom `is_active`, `defaultScope` tidak ditulis. Konsep dan perilaku runtime selengkapnya ada di [`catalogs/rdf/default-scope.md`](../../../catalogs/rdf/default-scope.md) dan [`features/default-scope`](../../../features/default-scope/README.md). Sinkronisasi nilai ini saat schema berubah ditangani oleh [`payload sync`](./sync.md#default-scope-is_active-built-in).

## Derivasi Soft-Delete dan Flag `--schema-path`

Bila tabel memiliki kolom soft-delete (`is_deleted`/`deleted_at`/`deleted_by`), `payload generate` membaca SDF dari `--schema-path` lalu menurunkan blok [`softDelete`](../../../catalogs/rdf/soft-delete.md) ke RDF beserta metadata FK-nya. Pembacaan SDF bersifat wajib karena base length kolom reusable hanya tercatat di SDF, tidak di database.

Yang dilakukan untuk tabel soft-delete:

1. Menulis blok `softDelete` RDF (`enabled` dan `reusable` dari SDF, `visibility` default `active_only`)
2. Men-set `fieldValidation[field].constraints.maxLength` kolom reusable ke base length (bukan physical length)
3. Mengeluarkan ketiga kolom soft-delete dari `fieldName` dan turunannya (kolom dikelola otomatis runtime)
4. Menulis metadata FK (`softDeleteFkChecks`, `softDeleteFkChildren`, `softDeleteCascadeTree`)

Field `softDeleteFkParents` (registry FK ke parent soft-delete) juga ditulis pada tabel anak non-soft-delete yang ber-FK ke tabel soft-delete. Untuk tabel tanpa kolom soft-delete, `--schema-path` diabaikan dan output identik dengan behavior tanpa fitur ini.

### Kondisi ERROR

Kehadiran satu saja kolom bernama soft-delete di tabel mewajibkan deklarasi SDF yang konsisten. Generate gagal dengan ERROR bila salah satu kondisi berikut terpenuhi:

| Kondisi | Pesan |
|---------|-------|
| `--schema-path` kosong | `... has soft-delete columns ... but --schema-path was not provided ...` |
| SDF gagal dimuat dari path tersebut | `... failed to load SDF from --schema-path='...'` |
| Tabel tidak dideklarasikan di SDF | `... has soft-delete columns but is not declared in the SDF at '...'` |
| Deklarasi tabel tanpa blok `softDelete` valid | `... its SDF declaration has no valid softDelete block (softDelete.enabled !== true)` |

Aturan ini disengaja (by-design): tanpa cek SDF, kolom soft-delete akan masuk RDF sebagai field writable biasa dan tabel diperlakukan sebagai tabel normal, sehingga `/delete` menjadi hard delete tanpa terlihat.

### Tabel Legacy dengan Kolom Bernama Soft-Delete

Tabel non-RESTForge yang kebetulan punya kolom bernama `is_deleted`, `deleted_at`, atau `deleted_by` (lengkap maupun parsial) terkena aturan yang sama: nama kolom tersebut adalah namespace yang direservasi kontrak soft-delete. Dua jalur mitigasi:

1. **Rename kolom** di database bila kolom tersebut bukan dimaksudkan sebagai soft-delete RESTForge, lalu generate ulang
2. **Lengkapi kontrak soft-delete**: tambahkan kolom yang kurang beserta CHECK konsistensi sesuai [spec SDF](../../../catalogs/sdf/soft-delete.md), deklarasikan tabel di SDF dengan blok `softDelete` valid, lalu jalankan generate dengan `--schema-path`

---

**Lihat juga**: [`payload/`](./) · [`commands/`](../) · [`README`](../../../README.md)
