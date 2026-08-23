# `payload generate`

> Generate file payload JSON berdasarkan introspeksi tabel database.

## Pattern

```
npx restforge payload generate --config=<FILE> --table=<NAME> [options]
```

## Flag Wajib

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--config <FILE>` | - | File config database (`.env`) |
| `--table <NAME>` | - | Nama tabel spesifik yang akan di-generate payload-nya |

## Flag Opsional

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--output <DIR>` | `payload/` | Output directory untuk file payload yang di-generate |
| `--schema-path <PATH>` | `schema` | Lokasi SDF (file atau folder). Wajib memuat deklarasi tabel bila tabel memiliki kolom soft-delete, karena blok `softDelete` RDF diturunkan dari SDF. Diabaikan untuk tabel non-soft-delete |
| `--detail <NAME>` | - | Nama tabel detail untuk generate master-detail otomatis. Lihat [Master-Detail Otomatis (`--detail`)](#master-detail-otomatis---detail) |

## Contoh

```bat
:: Generate untuk satu tabel ke folder default
npx restforge payload generate --config=db.env --table=users

:: Generate untuk satu tabel ke folder custom
npx restforge payload generate --config=db.env --table=users --output=./my-payloads

:: Generate tabel soft-delete dengan lokasi SDF eksplisit
npx restforge payload generate --config=db.env --table=category --schema-path=./schema

:: Generate master beserta blok masterDetail otomatis dari tabel detail
npx restforge payload generate --config=db.env --table=sales_order --detail=sales_order_item
```

## Field Validation Otomatis

Command `payload generate` melakukan introspeksi schema database lalu menurunkan `fieldValidation` untuk setiap field di output payload. Tujuannya menyediakan baseline validasi yang konsisten dengan constraint database sehingga endpoint `/create` dan `/update` menolak input invalid di application layer sebelum SQL dieksekusi.

### Sumber Introspeksi

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

### Customisasi Manual

Output `payload generate` adalah baseline. Constraint tambahan seperti `format`, `pattern`, `min`/`max`, `enum`, atau preset `email`/`phone`/`url`/`uuid` perlu ditambahkan secara manual di file payload setelah generate. Detail constraint per tipe ada di [`catalogs/rdf/field-validation`](../../../catalogs/rdf/field-validation.md).

## Default Scope Otomatis (`is_active`)

Bila tabel memiliki kolom `is_active`, `payload generate` secara otomatis menambahkan `defaultScope` built-in ke output payload:

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
2. Mengatur `fieldValidation[field].constraints.maxLength` kolom reusable ke base length (bukan physical length)
3. Mengeluarkan ketiga kolom soft-delete dari `fieldName` dan turunannya (kolom dikelola secara otomatis oleh runtime)
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

## Master-Detail Otomatis (`--detail`)

Flag `--detail=<table>` menghasilkan RDF master seperti biasa DITAMBAH blok [`masterDetail`](../../../catalogs/rdf/master-detail.md) terisi otomatis dari introspeksi tabel detail, sehingga pasangan header-detail tidak perlu ditulis manual dari nol.

### Yang Dihasilkan

Saat `--detail` diberikan, generate melakukan hal berikut:

1. Menulis blok `masterDetail` di level root RDF master:
   - `enabled: true`, `detailTable`, `foreignKey` (kolom FK di tabel detail yang merujuk primary key master), `cascadeDelete` (`true` bila FK dideklarasikan `ON DELETE CASCADE`), `transactionMode: "required"`.
   - `detailConfig.tableName`, `primaryKey`, `fieldName` (seluruh kolom tabel detail), `detailQuery` (menunjuk file SQL yang ikut ditulis), dan `requiredFields` (kolom `NOT NULL` tanpa default di tabel detail, di luar primary key dan kolom FK).
   - Kolom database GENERATED pada tabel detail diturunkan menjadi entry `detailConfig.autoCalculateFields` bertipe `"generated"`, lengkap dengan `formula` dari ekspresi kolom bila tersedia.
   - `detailConfig.fieldValidation`: tipe dan constraint tiap kolom detail, format sama dengan `fieldValidation` header. `detailConfig.foreignKeys`: daftar foreign key tabel detail yang bukan foreign key ke tabel master. Keduanya dipakai [`payload migrate`](./migrate.md#master-detail-menjadi-details) untuk menghasilkan blok `details[]` UDF secara penuh — lihat [`catalogs/rdf/master-detail.md`](../../../catalogs/rdf/master-detail.md).
2. Menulis file `payload/query/<detail-kebab>-detail.sql` berisi `SELECT` seluruh kolom detail, `WHERE <foreignKey> = $1`, dan `ORDER BY` kolom `line_number` bila kolom itu ada, atau primary key detail bila tidak.
3. Mengaktifkan `createComposite`, `updateComposite`, dan `readComposite` di blok `action` (action standar lainnya tidak berubah).
4. Tidak menulis file RDF standalone untuk tabel detail — tabel detail hanya dikelola lewat blok `masterDetail` pada RDF master.

Bagian yang **tetap perlu dilengkapi manual** setelah generate: `headerCalculations` dan entry `autoCalculateFields` bertipe `calculated` (formula per baris yang bukan kolom GENERATED database). RDF hasil generate menyertakan key `_manualStub` berisi petunjuk singkat cara mengisi keduanya — lihat [`catalogs/rdf/master-detail.md`](../../../catalogs/rdf/master-detail.md#pengisian-manual-setelah-generate---detail).

### Blok `masterDetail` Existing Dipertahankan

Bila file RDF master sudah ada dan sudah memiliki blok `masterDetail`, generate ulang dengan `--detail` **tidak menimpa** blok tersebut — blok lama dipertahankan apa adanya (konsisten dengan preservasi blok deklaratif lain saat regenerate), dan sebuah warning dicetak yang menjelaskan cara memaksa regenerasi dari database: hapus blok `masterDetail` dari file RDF, lalu jalankan ulang command yang sama.

### Kondisi ERROR

| Kondisi | Keterangan |
|---------|-----------|
| Tabel detail tidak ada di database | `--detail` menunjuk nama tabel yang tidak ditemukan |
| Tabel detail tidak memiliki foreign key ke tabel master | `masterDetail` mensyaratkan relasi FK detail → master |
| Primary key tabel detail bukan VARCHAR UUID-compatible | Primary key native `uuid`, atau `varchar`/`text` tanpa default auto-increment sequence |
| `--detail` bernilai sama dengan `--table` | Master dan detail tidak boleh tabel yang sama |
| Foreign key tabel detail tidak merujuk persis primary key tabel master | FK yang merujuk kolom master selain primary key ditolak dengan pesan yang menyebut nama FK, kolom lokal, kolom yang dirujuk, dan primary key master yang diharapkan |

Kondisi terakhir berlaku karena runtime composite mengisi kolom FK detail dengan nilai primary key header; FK yang merujuk kolom lain membuat semantik composite salah secara diam-diam saat runtime.

Bila tabel detail memiliki lebih dari satu foreign key yang merujuk tabel master, generate memilih kandidat pertama yang merujuk primary key master. Bila kandidat pertama tidak merujuk primary key tetapi ada kandidat lain yang merujuk, kandidat tersebut yang dipakai, disertai warning yang menjelaskan pergantian kandidat. Bila metadata kolom yang dirujuk tidak tersedia dari introspeksi untuk kandidat tertentu, pengecekan FK-merujuk-primary-key dilewati untuk kandidat itu dengan warning eksplisit (tidak ditebak). Warning juga dicetak untuk kasus kandidat lebih dari satu secara umum, agar blok `masterDetail` hasil generate diperiksa ulang secara manual bila relasi yang dimaksud bukan yang dipilih otomatis.

---

**Lihat juga**: [`payload/`](./) · [`commands/`](../) · [`README`](../../../README.md)
