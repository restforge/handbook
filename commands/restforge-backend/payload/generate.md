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
| `--schema-path <PATH>` | `schema` | Lokasi SDF (file atau folder). Wajib memuat deklarasi tabel bila tabel memiliki kolom soft-delete, karena blok `softDelete` RDF diturunkan dari SDF. Untuk tabel lain, SDF dipakai bila tersedia untuk menurunkan [`deleteReferences`](#rincian-perujuk-delete-deletereferences) serta [CHECK dan foreign key](#turunan-check-dan-foreign-key-dari-sdf) |
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

| Informasi | Data yang Diambil |
|---|---|
| Metadata kolom | `data_type`, `udt_name`, `column_default`, `is_nullable`, `character_maximum_length`, `numeric_precision`, `numeric_scale` |
| Constraint tabel | Daftar PRIMARY KEY dan UNIQUE constraint beserta kolom-kolomnya |

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

### Kustomisasi Manual

Output `payload generate` adalah baseline. Bila generate diulang dan isi file payload atau file query SQL di `payload/query/` berubah, versi lama diarsipkan lebih dulu ke `.restforge/archive/` (lihat [Arsip File Lama](../conventions.md#arsip-file-lama)).

Constraint tambahan seperti `format`, `pattern`, atau preset `email`/`phone`/`url`/`uuid` perlu ditambahkan secara manual di file payload setelah generate. `enum`, `min`, `max`, dan `notEqual` diturunkan otomatis bila kolomnya punya CHECK di SDF (lihat [Turunan CHECK dan Foreign Key dari SDF](#turunan-check-dan-foreign-key-dari-sdf)). Detail constraint per tipe ada di [`catalogs/rdf/field-validation`](../../../catalogs/rdf/field-validation.md).

## Generate Ulang pada RDF yang Sudah Ada

Bila file RDF target sudah ada, `payload generate` menggabungkan hasil introspeksi dengan isi file lama. Kustomisasi yang ditambahkan manual atau lewat `payload sync --expand-fk` tidak hilang.

Nilai berikut selalu diambil ulang dari database dan SDF:

- Kolom fisik tabel di `fieldName`, beserta `dateTimeFields` dan `uniqueConstraints`. Kolom yang sengaja dihapus dari `fieldName` tidak ditambahkan kembali (lihat [Kolom yang Dihapus dari `fieldName`](#kolom-yang-dihapus-dari-fieldname)).
- Constraint `fieldValidation` yang berasal dari struktur tabel, yaitu `primaryKey`, `autoGenerate`, `unique`, `default`, `scale`, `precision`, `enum`, `maxLength`, dan `required` dari kolom NOT NULL.
- Blok `softDelete` beserta metadata FK-nya, dan `deleteReferences`.
- Action standar yang belum ditetapkan di file lama. Tujuh action standar adalah `datatables`, `create`, `update`, `delete`, `first`, `lookup`, dan `read`.

Nilai berikut dipertahankan dari file lama:

- Blok yang tidak dihasilkan generator, misalnya `masterDetail` (termasuk `headerCalculations` dan `autoCalculateFields` di dalamnya), `workflow`, `components`, `authGuard`, dan `processor`.
- Nilai action standar yang sudah ditetapkan, misalnya `create: false` pada resource baca-saja.
- Flag `action` di luar tujuh action standar, misalnya `workflow`. Action composite tetap aktif selama RDF memiliki blok `masterDetail`, walaupun command dijalankan tanpa `--detail`.
- Aturan `fieldValidation` yang ditambahkan manual, misalnya `minLength`, `trim`, `format`, `pattern`, `hash`, dan `hashCost`. `maxLength` manual dipertahankan bila lebih kecil dari panjang kolom. Tipe manual juga dipertahankan selama masih sejenis dengan tipe kolom, misalnya `text` untuk kolom varchar.
- Referensi `file:` pada `datatablesQuery`, `viewQuery`, dan `exportQuery` yang menunjuk file selain `<table>-datatables.sql`, misalnya `file:query/<table>-join.sql`.
- SQL inline yang ditulis manual pada ketiga key tersebut, misalnya query JOIN atau query yang sengaja tidak memilih kolom password.
- Kolom JOIN di `fieldName`, misalnya `supplier_name`, selama kolom itu masih dipilih query yang dirujuk. Kolom JOIN ditempatkan di akhir `fieldName`.
- Entri `datatablesWhere` yang kolomnya masih ada. Kolom baru dari database disisipkan sebelum `"all"`.
- Filter `defaultScope` selain `is_active`.
- `auditColumns` bentuk object, selama semua kolom pemetaannya ada di tabel. Kolom audit kustom seperti `insert_date` tidak ditambahkan ke `fieldName`. Bila ada kolom pemetaan yang tidak ditemukan, generator memakai perilaku default dan menampilkan peringatan.
- File `<table>-datatables.sql` yang sudah diedit manual, misalnya ditambah `ORDER BY`. File yang belum diedit tetap ditulis ulang dari `fieldName`.

Generate ulang tanpa perubahan database menghasilkan file yang sama, termasuk urutan key-nya. Nilai yang dipertahankan dilaporkan pada baris `Preserved manual block(s)`:

```
Preserved manual block(s): masterDetail, workflow, action.create, action.workflow, datatablesQuery, viewQuery, fieldName.supplier_name, fieldValidation.password
```

File datatables yang diedit manual dilaporkan pada baris terpisah:

```
Query kept:    D:\projects\ecommerce\payload\query\product-datatables.sql (edited manually)
```

### Kolom Baru dan SQL JOIN

`payload generate` tidak mengubah file SQL JOIN. Kolom baru dari database masuk ke `fieldName`, tetapi belum dipilih SQL JOIN yang dirujuk RDF. Generator menampilkan peringatan beserta command untuk memperbarui SQL tersebut:

```
Warning: query/good-receipt-join.sql does not select column(s) receiver_name from fieldName. Run 'npx restforge payload sync --table=good_receipt --expand-fk=both' to refresh the JOIN query.
```

Command itu menulis ulang file SQL JOIN dengan kolom baru. Bila kolom tampilan dulu dipilih lewat `--fk-columns`, sertakan nilai yang sama agar pilihan kolom tidak berubah.

SQL inline manual dan file datatables yang diedit manual tidak ditulis ulang oleh command mana pun. Kolom baru yang tidak dipilih query tersebut tetap masuk ke `fieldName`, tetapi tidak ditambahkan ke `datatablesWhere`. Generator menampilkan peringatan untuk kolom itu:

```
Warning: inline datatablesQuery was kept and does not select new column(s) note from fieldName. Add them to the query manually if needed, and to datatablesWhere to make them searchable.
```

Tambahkan kolom tersebut ke query secara manual bila diperlukan. Agar kolom bisa dicari di datatables, tambahkan juga ke `datatablesWhere`.

### Kolom yang Dihapus dari `fieldName`

Kolom yang dihapus dari `fieldName`, misalnya `password`, tidak ditambahkan kembali oleh generate berikutnya. Kolom itu juga tidak masuk ke `datatablesWhere`, `fieldValidation`, dan file `<table>-datatables.sql`. Kolom yang benar-benar baru di tabel tetap ditambahkan otomatis.

Generator membedakan kedua jenis kolom tersebut dari daftar kolom tabel yang dicatat di folder `payload/.meta/`. Setiap RDF punya satu file dengan nama yang sama, misalnya `payload/.meta/users.json` untuk `payload/users.json`. Bila `--output` dipakai, folder `.meta` berada di dalam folder output tersebut. File di folder ini dikelola RESTForge dan tidak perlu diedit. Commit folder `.meta` bersama file RDF agar generate di komputer lain menghasilkan `fieldName` yang sama.

Untuk memakai kembali kolom yang sudah dihapus, tambahkan nama kolom itu ke `fieldName`. Generate dan sync berikutnya mempertahankan kolom tersebut.

RDF yang dibuat sebelum folder `.meta` tersedia belum punya daftar kolom. Pada RDF seperti ini, generator tidak bisa membedakan kolom baru dari kolom yang sengaja dihapus, sehingga tidak ada kolom yang ditambahkan. Generator menampilkan peringatan yang menyebut kolom tersebut:

```
Warning: No column snapshot found for users.json. Column(s) not in fieldName were not added: password, phone. Add them to fieldName manually if needed.
```

Tambahkan kolom yang memang diperlukan ke `fieldName`. Daftar kolom dibuat setelah command selesai, sehingga peringatan ini hanya muncul sekali.

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

Field `softDeleteFkParents` (registry FK ke parent soft-delete) juga ditulis pada tabel anak non-soft-delete yang ber-FK ke tabel soft-delete. Untuk tabel tanpa kolom soft-delete, derivasi soft-delete tidak dijalankan dan tidak ada blok `softDelete` yang ditulis.

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

## Rincian Perujuk Delete (`deleteReferences`)

Bila SDF di `--schema-path` dapat dimuat, `payload generate` mencari tabel lain yang mendeklarasikan relasi `belongsTo` ke tabel yang di-generate. Tabel anak yang relasinya menahan penghapusan ditulis ke `deleteReferences`, lengkap dengan kolom foreign key dan kolom yang dirujuknya.

```json
"deleteReferences": [
  { "table": "product", "column": "category_id", "references": "category_id" }
]
```

Relasi ber-`onDelete: cascade` dan `onDelete: setNull` tidak dimasukkan karena tidak pernah menolak penghapusan. Key ini hanya ditulis bila ada minimal satu tabel anak, sehingga RDF tabel tanpa anak tidak berubah.

Endpoint hasil `endpoint create` memakai `deleteReferences` untuk menyebut data perujuk beserta jumlahnya pada [response 409 `/delete`](../../../api-spec/endpoint-delete.md#409--foreign-key-constraint). RDF yang dibuat sebelum fitur ini ada baru mendapat `deleteReferences` setelah `payload generate` dijalankan ulang dengan `--schema-path`.

## Turunan CHECK dan Foreign Key dari SDF

Bila SDF di `--schema-path` dapat dimuat, `checks` tabel diturunkan ke `fieldValidation` dan ke registry [`checkConstraints`](../../../catalogs/rdf/check-constraints.md). Aturan pemetaannya ada di [`rdf/field-validation.md`](../../../catalogs/rdf/field-validation.md#turunan-check-sdf).

```javascript
checks: [
  { field: 'status', in: ['waiting', 'called', 'done'] },
  { field: 'deposit_value', gt: 0 },
  { field: 'channel', neq: 'none' }
]
```

```json
{ "name": "deposit_value", "type": "number", "constraints": { "scale": 2, "precision": 20, "required": true, "min": 0.01 } },
{ "name": "status", "type": "string", "constraints": { "required": true, "maxLength": 20, "default": "waiting", "enum": ["waiting", "called", "done"] } },
{ "name": "channel", "type": "string", "constraints": { "required": true, "maxLength": 20, "notEqual": "none" } }
```

Untuk Oracle dan SQLite, relasi `belongsTo` juga diturunkan ke [`foreignKeyConstraints`](../../../catalogs/rdf/foreign-key-constraints.md). PostgreSQL dan MySQL tidak memerlukannya karena error driver kedua dialect itu sudah menyebut kolom foreign key.

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
