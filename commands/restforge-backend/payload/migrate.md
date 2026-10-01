# `payload migrate`

> Konversi file payload backend (RDF) menjadi payload frontend (UDF) untuk RESTForge Designer.

Output selalu berbentuk SPLIT multi-file di dalam sebuah output directory, bukan satu
file UDF tunggal. Migrator juga melakukan auto-discovery tabel JOIN sehingga satu RDF
utama ber-JOIN dapat menghasilkan beberapa page sekaligus. Detail kedua perilaku ini
diuraikan pada bagian [Output Split Multi-File](#output-split-multi-file) dan
[Auto-Discovery JOIN](#auto-discovery-join).

## Pattern

```
npx restforge payload migrate --name=<STRING> --project=<STRING> [options]
```

## Flag Wajib

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--name <STRING>` | - | Nama file payload backend (mis. `visitors.json`). Path resolusi: relatif terhadap `cwd` atau `cwd/payload/` |
| `--project <STRING>` | - | Nama project (kebab-case). Dipakai sebagai segment path di `apiBaseUrl`: `http://{host}:{port}/api/{project}` |

## Flag Opsional

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--output <STRING>` | `frontend/payload/` | Output directory tempat file split ditulis. Jika nilai diakhiri `.json`, direktori induknya dipakai sebagai output directory (file tunggal tidak lagi dihasilkan) |
| `--config <STRING>` | `null` | Database config file (`.env`). Dibaca untuk ambil `SERVER_ADDRESS` dan `SERVER_PORT` (backend port). Fallback ke default config (`restforge config set-default`) jika tidak diisi |
| `--app-name <STRING>` | diturunkan dari `--project` (Title Case) | Nama aplikasi yang dipakai di `appConfig.appName` UDF output. Jika tidak diisi, di-derive dari `--project` dengan format Title Case, mis. `visitors-app` menjadi `"Visitors App"` |
| `--app-code <STRING>` | mengikuti `--project` | Kode aplikasi kebab-case yang dipakai di `appConfig.appCode` UDF output. Dipakai juga sebagai nama file aggregator (`<appCode>.json`) |
| `--plugin <STRING>` | `"vanilla-js-basic"` | Plugin ID Designer yang ditulis di `appConfig.plugin` UDF output |
| `--port <NUMBER>` | `8000` | Port aplikasi frontend yang ditulis ke `appConfig.port`. Independen dari backend port yang dipakai di `apiBaseUrl` |
| `--overwrite` | `false` | Buat ulang file page di `pages\` yang sudah ada dari nol sehingga perubahan manual pada page hilang, serta timpa `app-config.json` bila aggregator belum ada. Tanpa flag ini, page yang sudah ada digabung (lihat [Migrate Ulang Page yang Sudah Ada](#migrate-ulang-page-yang-sudah-ada)). Aggregator dan `app-config.json` milik aplikasi yang sudah ada selalu digabung. File yang isinya berubah diarsipkan lebih dulu ke `.restforge/archive/` |

## Contoh

### Migrasi Standar dengan Config Database

```bat
:: Working directory: backend project root
npx restforge payload migrate --name=visitors.json --output=..\frontend\payload --config=db-connection.env --project=visitors-app
```

Output untuk RDF `visitors.json` yang ber-JOIN ke `visitor_categories` (auto-discovery
menghasilkan dua page):

```
============================================================
PAYLOAD MIGRATE - RDF (backend) -> UDF (frontend, split)
============================================================

  Input        : D:\workspace\...\backend\payload\visitors.json
  Output dir   : D:\workspace\...\frontend\payload
  Project      : visitors-app
  apiBaseUrl   : http://127.0.0.1:3000/api/visitors-app
  Backend port : 3000
  Frontend port: 8000
  Homepage     : visitors
  Pages        : 2

  [OK] visitor-categories: 6 field(s), 0 table column(s)
  [OK] visitors: 4 field(s), 4 table column(s)

  Files written:
    - app-config.json
    - pages\visitor-categories.json
    - pages\visitors.json
    - visitors-app.json

  Migration completed successfully.
```

RDF tanpa JOIN menghasilkan `Pages: 1` dengan tetap memakai struktur split yang sama
(`app-config.json`, satu file di `pages\`, dan aggregator `<appCode>.json`).

### Migrasi Default (Output dan Config Auto)

```bat
:: Output otomatis ke frontend/payload/, config dari default config
npx restforge payload migrate --name=visitors.json --project=visitors-app
```

### Membuat Ulang Page dengan `--overwrite`

```bat
npx restforge payload migrate --name=visitors.json --project=visitors-app --overwrite
```

`--overwrite` membuat ulang page dari nol sesuai hasil konversi RDF, sehingga perubahan
manual pada page hilang. Tanpa flag ini, page yang sudah ada digabung dan perubahan
manual tetap ada (lihat [Migrate Ulang Page yang Sudah Ada](#migrate-ulang-page-yang-sudah-ada)).
Menambah tabel baru ke aplikasi yang sudah ada juga tidak memerlukan `--overwrite` (lihat
[Menambah Tabel ke Aplikasi yang Sudah Ada](#menambah-tabel-ke-aplikasi-yang-sudah-ada)).

Tanpa `--overwrite`, command hanya gagal bila `app-config.json` sudah ada tanpa aggregator,
atau bila file page yang sudah ada tidak bisa dibaca.

Dengan `--overwrite`, setiap file UDF yang isinya berubah diarsipkan lebih dulu ke
`.restforge/archive/<YYYYMMDD-HHMMSS>/` dengan path aslinya, termasuk halaman yang sudah
di-edit manual. Menyalin balik folder run tersebut membatalkan migrate (lihat
[Arsip File Lama](../conventions.md#arsip-file-lama)).

### Migrasi dengan App Metadata Custom

```bat
npx restforge payload migrate --name=contact.json --project=contact-mgmt --app-name="Contact Management" --app-code=contact-mgmt --plugin=vanilla-js-auth --port=8080
```

`--app-code=contact-mgmt` membuat aggregator ditulis sebagai `contact-mgmt.json`.
`--port=8080` menetapkan `appConfig.port` ke `8080` (backend port di `apiBaseUrl` tetap
dibaca dari config database).

## Resolusi Path Input

Argumen `--name` dapat berupa:

| Bentuk | Resolusi |
|--------|---------|
| `visitors.json` | Dicari di `cwd/visitors.json`, jika tidak ada di `cwd/payload/visitors.json` |
| `payload/visitors.json` | Path eksplisit relatif terhadap `cwd` |
| `D:\path\to\visitors.json` | Path absolut |

## Resolusi `apiBaseUrl`

Field `appConfig.apiBaseUrl` di UDF output dibentuk dari backend host dan backend port:

```
http://{SERVER_ADDRESS}:{SERVER_PORT}/api/{project}
```

| Source `SERVER_ADDRESS` dan `SERVER_PORT` | Trigger |
|-------------------------------------------|---------|
| File `--config <FILE>` | Jika flag dipakai |
| Default config (`restforge config set-default`) | Fallback jika `--config` tidak dipakai |
| `127.0.0.1:3000` | Default jika `SERVER_ADDRESS`/`SERVER_PORT` tidak terisi di config |

Backend port pada `apiBaseUrl` terpisah dari frontend port (`appConfig.port`) yang
diatur via `--port` (default `8000`).

Alamat loopback ini tidak dipakai apa adanya oleh frontend hasil generate. Browser
memilih path relatif saat halaman dibuka lewat reverse proxy, lihat
[Base URL di Belakang Reverse Proxy](../../../catalogs/udf/app-config.md#base-url-di-belakang-reverse-proxy).

## Output Split Multi-File

Migrate menulis beberapa file ke dalam output directory, bukan satu file UDF tunggal.
Struktur yang dihasilkan:

| File | Isi |
|------|-----|
| `app-config.json` | Blok `appConfig` bersama: `appName`, `appCode`, `plugin`, `apiBaseUrl`, `port`, `numberFormat`, `dateFormat`, `dateTimeFormat` |
| `pages\<pageId>.json` | Satu file per page, dibungkus `{ "pages": [ <page> ] }`. `<pageId>` adalah kebab-case nama tabel |
| `<appCode>.json` | Aggregator: `extends` ke `app-config.json`, `homepage`, daftar `include` setiap page, dan `navigation` |

Contoh isi `app-config.json`:

```json
{
  "appConfig": {
    "appName": "Visitors App",
    "appCode": "visitors-app",
    "plugin": "vanilla-js-basic",
    "apiBaseUrl": "http://127.0.0.1:3000/api/visitors-app",
    "port": 8000,
    "numberFormat": { "locale": "en-US", "currencyPrefix": "" },
    "dateFormat": "dd/MM/yyyy",
    "dateTimeFormat": "dd/MM/yyyy HH:mm:ss"
  }
}
```

`dateFormat` dan `dateTimeFormat` disalin dari `DATEFORMAT` dan `DATETIMEFORMAT` di file config
(`--config`). Bila keduanya kosong, default platform yang ditulis (`yyyy-MM-dd` dan
`yyyy-MM-dd HH:mm:ss.SSS`). Pola yang tidak didukung juga jatuh ke default, disertai warning.
Karena nilainya disalin saat migrate, setiap perubahan `DATEFORMAT` atau `DATETIMEFORMAT` wajib
diikuti `payload migrate` dan generate frontend ulang. Lihat
[`catalogs/udf/app-config.md`](../../../catalogs/udf/app-config.md#pola-tanggal-dateformat-dan-datetimeformat).

Contoh isi aggregator `visitors-app.json`:

```json
{
  "extends": "app-config.json",
  "homepage": "visitors",
  "pages": [
    { "include": "pages/visitor-categories.json" },
    { "include": "pages/visitors.json" }
  ],
  "navigation": {
    "items": [
      { "type": "page", "pageRef": "visitor-categories", "label": "Visitor Categories" },
      { "type": "page", "pageRef": "visitors", "label": "Visitors" }
    ]
  }
}
```

### Menambah Tabel ke Aplikasi yang Sudah Ada

Migrate tetap dijalankan satu RDF per perintah. Aplikasi multi-page disusun dengan menjalankan
migrate untuk setiap tabel ke output directory dan `--project` yang sama:

```bat
:: Tabel pertama: app-config.json, pages\visitors.json, dan visitors-app.json dibuat
npx restforge payload migrate --name=visitors.json --project=visitors-app --output=..\frontend\payload --config=db-connection.env

:: Tabel berikutnya: page baru ditulis, aplikasi yang ada ditambah
npx restforge payload migrate --name=departments.json --project=visitors-app --output=..\frontend\payload --config=db-connection.env
```

Migrate menganggap aplikasi sudah ada bila `app-config.json` dan aggregator `<appCode>.json`
sudah ada di output directory. Pada kondisi ini:

| File | Perlakuan |
|------|-----------|
| `pages\<pageId>.json` | Page baru ditulis. Page yang sudah ada, termasuk page tabel terkait hasil auto-discovery JOIN, digabung tanpa menghapus perubahan manual (lihat [Migrate Ulang Page yang Sudah Ada](#migrate-ulang-page-yang-sudah-ada)) |
| `<appCode>.json` | Digabung. Entri `include` untuk page baru ditambahkan tanpa duplikasi. Susunan `navigation` yang ada dipertahankan utuh, termasuk group bersarang, separator, item `link`, serta label dan icon hasil edit manual. Page baru masuk ke tingkat teratas `navigation` hanya bila `pageRef`-nya belum ada di tingkat mana pun, lalu dapat dipindahkan ke group yang sesuai. `homepage` dipertahankan |
| `app-config.json` | Digabung. Nilai `appConfig` yang sudah ada dipertahankan dan properti yang belum ada ditambahkan. Pengecualiannya `dateFormat` dan `dateTimeFormat`, yang selalu ditulis ulang dari `DATEFORMAT` dan `DATETIMEFORMAT` di file config agar sama dengan backend |

Aturan gabungan aggregator dan `app-config.json` berlaku dengan maupun tanpa `--overwrite`.
Output ditandai `(merged)` pada daftar `Files written`.

Bila `appConfig.appCode` di `app-config.json` yang ada berbeda dengan `--app-code` (atau
`--project`), command berhenti sebelum menulis file apa pun:

```
Error: app-config.json in <output> belongs to app 'visitors-app', but this run targets app 'other-app'.
Use a different --output for app 'other-app', or pass --project/--app-code=visitors-app to add the page to the existing app.
```

### Migrate Ulang Page yang Sudah Ada

Page yang sudah ada di `pages\` digabung dengan hasil konversi terbaru, sehingga
perubahan manual pada page tetap ada. Perubahan RDF, misalnya kolom baru, ikut masuk
tanpa `--overwrite`. Aturan ini juga berlaku untuk page tabel terkait yang ikut ditulis
karena [Auto-Discovery JOIN](#auto-discovery-join).

Contoh: page `category` sudah diedit manual.

```json
{
  "pages": [
    {
      "pageId": "category",
      "pageTitle": "Product Category",
      "pageIcon": "folder",
      "fields": [
        { "name": "category_name", "label": "Name", "type": "text", "width": "300px" }
      ],
      "fieldRows": [
        { "fields": ["category_name"] }
      ]
    }
  ]
}
```

Kolom `description` lalu ditambahkan ke tabel, RDF diperbarui dengan
[`payload sync`](./sync.md), dan migrate dijalankan ulang:

```bat
npx restforge payload migrate --name=category.json --project=probe
```

`pageTitle`, `pageIcon`, `label`, dan `width` tetap bernilai hasil edit manual. Field
`description` ditambahkan ke `fields` dan menjadi baris baru di `fieldRows`:

```json
"fieldRows": [
  { "fields": ["category_name"] },
  { "fields": ["description"] }
]
```

Output menandai page yang digabung dengan `(merged)`, file yang isinya tidak berubah
dengan `(unchanged)`, dan menyebut jumlah nilai manual yang dipertahankan per page:

```
  Files written:
    - app-config.json  (unchanged)
    - pages\category.json  (merged)
    - probe.json  (unchanged)

  Kept customization(s):
    - category: 4 value(s)
```

File yang isinya tidak berubah tidak ditulis ulang.

#### Aturan Penggabungan

Setiap nilai pada page dibandingkan dengan hasil konversi terakhir yang dicatat migrate.

| Kondisi | Hasil |
|---|---|
| Nilai tidak diubah manual | Mengikuti hasil konversi terbaru |
| Nilai diubah manual, RDF tidak berubah | Nilai manual dipertahankan |
| Nilai diubah manual dan juga berubah di RDF | Nilai manual dipertahankan, kecuali properti pada tabel berikutnya |

Menghapus properti dari page juga terhitung perubahan manual. Blok yang tidak pernah
ditulis migrate, seperti `fieldRows`, `fieldStates`, `workflow`, `workflowActions`,
`pageGroup`, dan `pageSubject`, dipertahankan utuh.

Beberapa properti terhubung langsung dengan backend, sehingga tetap mengikuti RDF:

| Properti | Hasil |
|---|---|
| `apiPath`, `primaryKey`, `type`, `required`, `maxlength`, `decimalPlaces`, `tableField`, `dataSource.type`, `dataSource.resource`, dan daftar `value` pada `dataSource.options` | Bila diubah manual dan juga berubah di RDF, nilai RDF dipakai disertai peringatan |
| `actions.<aksi>` bernilai `false` di hasil konversi | Selalu `false`. Peringatan tampil bila page menulis `true` |
| `temporalType` | Selalu mengikuti RDF |
| `dataSource.select` | Mengikuti RDF bila berubah di RDF. Kolom yang dirujuk `autofill` dan `optionColumns` selalu ditambahkan (lihat [Atribut Lookup Saat Migrate Ulang](#atribut-lookup-saat-migrate-ulang)) |

Contoh peringatan saat `decimalPlaces` diubah menjadi `0` di page, sedangkan RDF
mengubahnya menjadi `4`:

```
  Warnings:
    - product.fields.price.decimalPlaces changed in both UDF and RDF; RDF value 4 is used (UDF had 0)
```

#### Field Baru dan Field yang Dihapus

| Kondisi | Hasil |
|---|---|
| Kolom baru di RDF | Ditambahkan ke `fields` setelah field yang mendahuluinya, mendapat `tableOrder` berikutnya, dan ditambahkan sebagai baris baru di akhir `fieldRows` bila page memakainya |
| Field dihapus manual dari page | Tetap tidak ada |
| Kolom hilang dari RDF | Dikeluarkan dari `fields`, `fieldRows`, `fieldStates`, dan `features.dataFilters`, disertai peringatan |
| Field tambahan yang ditulis manual | Dipertahankan |

Aturan yang sama berlaku untuk field grid detail di `details[].fields[]`.

#### Folder `.meta`

Migrate mencatat hasil konversi terakhir setiap page di
`<output>/.meta/pages/<pageId>.json`, misalnya `frontend/payload/.meta/pages/category.json`.
Catatan ini menjadi pembanding pada migrate berikutnya. File di folder ini dikelola
RESTForge dan tidak perlu diedit. Commit folder `.meta` bersama file page agar migrate di
komputer lain menghasilkan penggabungan yang sama. `restforge-designer` tidak membaca
folder ini.

Page yang dibuat sebelum folder `.meta` tersedia belum punya catatan tersebut. Pada page
seperti ini, nilai yang berbeda dari hasil konversi dianggap perubahan manual, kecuali
properti yang tetap mengikuti RDF. Kolom baru tidak ditambahkan karena tidak bisa
dibedakan dari field yang sengaja dihapus. Migrate menampilkan peringatan yang menyebut
field tersebut:

```
  Warnings:
    - No snapshot found for page 'category'. Field(s) not added: description. Add them to the page manually if needed.
```

Tambahkan field yang diperlukan ke page secara manual. Catatan dibuat setelah command
selesai, sehingga peringatan ini hanya muncul sekali.

#### File Page yang Rusak

Bila file page tidak bisa dibaca, misalnya karena JSON tidak valid, migrate berhenti
sebelum menulis file apa pun:

```
Error: Existing page file ...\pages\category.json could not be read (Expected double-quoted property name in JSON at position 86 (line 5 column 38)).
Fix the file, or use --overwrite to recreate the page from scratch.
```

Perbaiki file lalu jalankan migrate ulang agar perubahan manual tetap ada. Cara lain,
jalankan dengan `--overwrite` untuk membuat ulang page dari nol. Dengan cara ini
perubahan manual di page tersebut hilang, dan file lama tersimpan di arsip.

### Penentuan Output Directory

| Bentuk `--output` | Output directory |
|-------------------|------------------|
| Tidak diisi | `frontend/payload/` (relatif terhadap `cwd`) |
| `..\frontend\payload` | Direktori tersebut |
| `..\frontend\payload\visitors-app.json` | Direktori induknya (`..\frontend\payload`); suffix `.json` tidak lagi berarti file tunggal |

Karena nama file aggregator ditentukan oleh `--app-code` (atau `--project`), mengakhiri
`--output` dengan `.json` tidak mengubah nama file output. Nilai berakhiran `.json` hanya
diturunkan ke direktori induknya sebagai root split.

## Auto-Discovery JOIN

Bila `datatablesQuery` pada RDF utama memuat JOIN, migrator memuat RDF tabel terkait dari
sibling folder payload dan mengonversinya menjadi page tersendiri. Satu RDF utama
ber-JOIN dapat menghasilkan beberapa page sekaligus.

Mekanisme:

1. Migrator mengurai JOIN pada `datatablesQuery` RDF utama (referensi `file:` di-resolve
   lebih dulu).
2. Untuk setiap tabel yang di-JOIN, RDF terkait dicari di folder payload yang sama. Nama
   file dicoba dalam urutan kebab-case (`visitor-categories.json`) lalu snake_case
   (`visitor_categories.json`).
3. RDF terkait dikonversi menjadi page. Urutan page: tabel master (related) lebih dulu,
   RDF utama menjadi page terakhir dan ditetapkan sebagai `homepage` di aggregator.
4. Bila RDF terkait tidak ditemukan, page tidak di-generate dan sebuah warning
   dikumpulkan. Referensi `select` pada field FK tetap dipertahankan.

Contoh: migrate `visitors.json` yang JOIN ke `visitor_categories` menghasilkan
`Pages: 2` (`visitor-categories` sebagai master, `visitors` sebagai homepage).

Bila page tabel terkait sudah ada, page itu digabung dengan aturan
[Migrate Ulang Page yang Sudah Ada](#migrate-ulang-page-yang-sudah-ada), sehingga perubahan
manual pada page tersebut tetap ada.

## Aturan Konversi RDF → UDF

Migrator menginferensi struktur UDF dari struktur RDF berdasarkan aturan deterministic.
Detail mapping field type, exclude rules, dan inferensi label ada di [`catalogs/udf/`](../../../catalogs/udf/).

Ringkasan:

| Aspek | Behavior |
|-------|----------|
| Tipe field | Auto-detect dari `fieldValidation` RDF (boolean → checkbox, number → number, string enum → select static, `*_date` → date, dst.) |
| Field `date`/`timestamp` (`temporalType`) | Kolom `timestamptz` menjadi field `timestamp` dengan `temporalType: "timestamptz"`. Pola nilai tidak ditulis per field karena frontend membaca `appConfig.dateFormat` dan `appConfig.dateTimeFormat` |
| Field `number` (`format`, `decimalPlaces`) | `format: "number"`, atau `"currency"` bila RDF `constraints.format: "currency"` pada field itu, tidak pernah ditebak dari nama field. `decimalPlaces` dari `constraints.scale` RDF, fallback ke `constraints.precision` bila `scale` tidak ada (RDF lama), disertai warning per page |
| FK + JOIN → select | Field FK yang menjadi local column sebuah JOIN dikonversi menjadi dropdown `select` dengan `dataSource` API |
| JOIN discovery | RDF utama ber-JOIN memuat RDF tabel terkait dari sibling folder dan mengonversinya menjadi page tambahan (1 RDF utama → multi page) |
| Field di-exclude | Primary key (`constraints.primaryKey: true`), audit columns (`created_at`, `created_by`, `updated_at`, `updated_by`), display column milik tabel JOIN |
| Properti page | `tableName` → `pageId` (kebab-case), `pageTitle` (Title Case), `apiPath` (sama dengan `pageId`) |
| Fitur otomatis (search) | `datatablesWhere` memuat kolom searchable → `enableSearch: true` |
| Fitur otomatis (status filter) | Field status utama — `is_active` boolean ATAU `is_active`/`status`/`state` dengan enum RDF — → `enableStatusFilter: true` + `statusFilter`, dan field tersebut diberi `statusBadge: true` (lihat [`catalogs/udf/features.md`](../../../catalogs/udf/features.md#enablestatusfilter-dan-statusfilter)) |
| Fitur otomatis (data filter) | Field FK (JOIN) dan field enum lain (bukan field status utama) → `enableDataFilter: true` + `dataFilters[]` otomatis (lihat [`catalogs/udf/features.md`](../../../catalogs/udf/features.md#enabledatafilter-dan-datafilters)) |
| Fitur otomatis (filter toolbar) | Maksimal 2 filter (gabungan `statusFilter` + `dataFilters[]`) tampil inline, sisanya masuk filter button — detail lengkap di [`catalogs/udf/features.md`](../../../catalogs/udf/features.md#maks-2-filter-inline--filter-button) |
| Label dan placeholder | Auto-generate dari nama field |
| Master-detail (`masterDetail`) | RDF ber-`masterDetail` dengan `detailConfig.fieldValidation` dikonversi penuh menjadi blok `details[]` — lihat [Master-Detail Menjadi `details[]`](#master-detail-menjadi-details) |

### FK + JOIN menjadi `select`

Field FK yang muncul sebagai local column pada JOIN `datatablesQuery` otomatis menjadi
dropdown `select` dengan `dataSource` bertipe `api`:

```json
{
  "name": "category_id",
  "label": "Category",
  "type": "select",
  "inTable": true,
  "tableOrder": 6,
  "tableField": "category_name",
  "dataSource": {
    "type": "api",
    "resource": "visitor-categories",
    "select": ["category_id", "category_name"]
  }
}
```

| Properti | Asal |
|----------|------|
| `dataSource.resource` | Kebab-case nama tabel JOIN (`visitor_categories` → `visitor-categories`), cocok dengan apiPath endpoint `lookup` |
| `dataSource.select` | `[remoteColumn, displayColumn]`, mis. `["category_id", "category_name"]` |
| `tableField` | Display column dari JOIN (`category_name`), agar DataTable menampilkan nama, bukan UUID |

Display column diambil dari kolom JOIN aktual yang dipilih di SELECT, bukan tebakan
`<table>_name`, sehingga `select` dan `tableField` konsisten.

### Field Status Enum Menjadi `statusFilter`

Field bernama `is_active`, `status`, atau `state` yang beresolusi boolean (checkbox) ATAU
select+enum (`constraints.enum` di RDF) otomatis menjadi field status utama:
`enableStatusFilter: true` dengan `statusFilter.options` sesuai tipenya. Bila lebih
dari satu field cocok, field yang lebih dulu muncul di `fieldName` yang dipakai. Untuk
field enum, `options` diisi nilai enum RDF apa adanya — BUKAN dikonversi ke
Active/Inactive seperti pada field boolean.

RDF (field `status`, enum 3 nilai):

```json
{
  "name": "status",
  "type": "string",
  "constraints": {
    "default": "waiting",
    "enum": ["waiting", "called", "done"]
  }
}
```

UDF hasil migrate:

```json
"features": {
    "enableStatusFilter": true,
    "statusFilter": {
        "field": "status",
        "label": "Status",
        "options": [
            {"value": "waiting", "text": "Waiting"},
            {"value": "called", "text": "Called"},
            {"value": "done", "text": "Done"}
        ]
    }
}
```

Field status utama juga diberi `statusBadge: true`, sehingga tampil sebagai kolom
Status berpalet tetap (lihat [`catalogs/udf/field-types.md`](../../../catalogs/udf/field-types.md#kolom-status-statusbadge)).
Penandaan ini bisa dilepas dengan menulis `statusBadge: false` pada field di UDF.
Nilai `false` tersebut dipertahankan saat migrate ulang tanpa `--overwrite`.

Field FK (JOIN) dan field enum lain yang BUKAN field status utama masuk
`dataFilters[]`, bukan `statusFilter` — lihat
[`catalogs/udf/features.md`](../../../catalogs/udf/features.md#enabledatafilter-dan-datafilters)
untuk detail lengkap dan contoh.

### Master-Detail Menjadi `details[]`

RDF dengan blok [`masterDetail`](../../../catalogs/rdf/master-detail.md) dikonversi menjadi
blok [`details[]`](../../../catalogs/udf/master-detail.md) pada page UDF, asalkan
`detailConfig` sudah memuat `fieldValidation` — hasil `payload generate --detail=<table>`
(lihat [Master-Detail Otomatis](./generate.md#master-detail-otomatis---detail)). RDF lama
yang `detailConfig`-nya belum memuat `fieldValidation` (ditulis manual sebelum command ini
mendukung info tipe kolom detail) hanya mengonversi field header, disertai warning, dan
page yang dihasilkan tidak memiliki blok `details[]` sama sekali — jalankan `payload
generate --detail=<table>` untuk melengkapi `detailConfig`, lalu migrate ulang.

Pemetaan RDF → UDF untuk detail:

| Aspek | Sumber RDF | Hasil UDF |
|---|---|---|
| `detailId` | `detailConfig.tableName` (nama tabel apa adanya, tanpa transformasi kebab-case) | `details[].detailId` |
| `detailTitle` | `detailConfig.tableName` dalam Title Case, misalnya `sales_order_item` menjadi `Sales Order Item` | `details[].detailTitle` |
| `primaryKey` | `detailConfig.primaryKey` | `details[].primaryKey` |
| Daftar field | `detailConfig.fieldName`, dikurangi primary key detail, kolom audit, foreign key ke master, dan kolom penanda urutan baris (`line_number`) | `details[].fields[]` |
| Tipe field | `detailConfig.fieldValidation` | `type`, sama aturan auto-detect dengan field header (lihat tabel di atas) |
| Field wajib | `detailConfig.requiredFields` | `required: true` |
| Foreign key non-master | `detailConfig.foreignKeys` | field `select` dengan `dataSource` API — `resource` dari tabel yang dirujuk, `select` berisi kolom yang dirujuk dan kolom tampilan, `url` diisi eksplisit ke endpoint lookup resource tersebut |
| Kalkulasi per baris (`autoCalculateFields`) | Formula pola `kolomA * kolomB` | field `readonly: true` dengan `calculated.formula` dan `format: "number"`. Formula di luar pola perkalian dua kolom (penjumlahan, fungsi SQL, lebih dari dua operand) tetap dikonversi jadi `readonly: true` tanpa `calculated`, disertai warning per kolom |
| Agregat header (`headerCalculations`) | `masterDetail.headerCalculations` | blok `summary` pada `details[]` (lihat [`catalogs/udf/master-detail.md`](../../../catalogs/udf/master-detail.md#properti-summary)). Tidak ditulis sama sekali bila `headerCalculations` tidak ada |

Seluruh field detail mendapat `inTable: true` dengan `tableOrder` kontigu mengikuti urutan
kemunculan di `detailConfig.fieldName` — berbeda dari field header yang hanya `inTable`
bila ada di `datatablesQuery`, grid detail selalu menampilkan seluruh kolomnya.

Contoh hasil konversi untuk detail item sales order:

```json
"details": [
    {
        "detailId": "sales_order_item",
        "detailTitle": "Sales Order Item",
        "primaryKey": "sales_order_item_id",
        "fields": [
            {
                "name": "product_id",
                "label": "Product",
                "type": "select",
                "required": true,
                "inTable": true,
                "tableOrder": 1,
                "tableField": "product_name",
                "dataSource": {
                    "type": "api",
                    "resource": "product",
                    "url": "product/lookup",
                    "select": ["product_id", "product_name"]
                }
            },
            {
                "name": "quantity",
                "label": "Quantity",
                "type": "number",
                "required": true,
                "inTable": true,
                "tableOrder": 2
            },
            {
                "name": "price",
                "label": "Price",
                "type": "number",
                "inTable": true,
                "tableOrder": 3
            },
            {
                "name": "total_amount",
                "label": "Total Amount",
                "type": "number",
                "inTable": true,
                "tableOrder": 4,
                "readonly": true,
                "calculated": { "formula": "quantity * price" },
                "format": "number"
            }
        ],
        "summary": {
            "totalItems": true,
            "totalQtyField": "quantity",
            "grandTotalField": "total_amount"
        }
    }
]
```

#### Status `line_number`

`line_number` dikecualikan dari field form detail seperti pada tabel pemetaan di atas. Kolom tersebut bertujuan menyimpan urutan bisnis item per header, bukan sekadar nomor tampilan `#`. Aplikasi yang hanya membutuhkan tampilan terurut berdasarkan primary key atau kolom lain tidak wajib menambahkannya.

Server belum mengisi `line_number` secara otomatis. Bila tabel detail memiliki kolom `line_number` `NOT NULL` tanpa default, halaman hasil migrate gagal menyimpan data karena kolom tersebut tidak dikirim. Field `line_number` dapat ditambahkan manual ke `details[].fields[]` setelah migrate, dengan konsekuensi yang dijelaskan di [Menyimpan Urutan dengan Field Angka Manual](../../../catalogs/udf/master-detail.md#menyimpan-urutan-dengan-field-angka-manual). Lihat juga [tujuan `line_number` di RDF](../../../catalogs/rdf/master-detail.md#tujuan-line_number-urutan-bisnis-tersimpan).

#### Atribut Lookup Saat Migrate Ulang

[Atribut lookup tambahan](../../../catalogs/udf/data-source.md#atribut-lookup-tambahan-autofill-optioncolumns-lookupdisplay)
`autofill`, `optionColumns`, dan `lookupDisplay` ditulis manual setelah migrate. Migrate
ulang mempertahankan ketiganya sebagai perubahan manual, baik pada field halaman master
maupun pada field grid detail. `defaultValue` yang ditulis manual juga dipertahankan.

Kolom yang dirujuk `autofill` dan `optionColumns` (mis. `price`) ditambahkan ke
`dataSource.select` bila belum ada, sehingga endpoint lookup tetap mengirim kolom yang
dibutuhkan. Kolom tersebut ditambahkan di belakang tanpa mengubah urutan kolom yang sudah
ada. Bila daftar kolom `select` berubah di RDF, daftar baru dipakai lalu ditambah kolom
rujukan tadi.

Migrate dengan `--overwrite` membuat ulang page dari nol, sehingga atribut ini perlu
ditulis ulang atau disalin dari arsip.

#### Kolom Agregat Header Disembunyikan Otomatis

Field header yang namanya cocok dengan salah satu key `masterDetail.headerCalculations`
mendapat `editorMode: "hidden"` dan `defaultValue: 0` secara otomatis pada hasil migrate —
kolom ini dihitung ulang oleh backend saat create/update composite, sehingga tidak perlu
diisi lewat form. Atribut lain field tersebut (`label`, `type`, `inTable`, `tableOrder`)
tetap mengikuti hasil konversi normal. Perilaku ini berlaku baik saat detail berhasil
dikonversi penuh menjadi `details[]` maupun saat migrate jatuh ke fallback header-only,
karena keduanya sama-sama membaca nama kolom langsung dari `masterDetail.headerCalculations`.

## Batasan

| Aspek | Behavior |
|-------|----------|
| Master-detail (`masterDetail`) | Dikonversi penuh menjadi `details[]` bila `detailConfig` sudah memuat `fieldValidation` (lihat [Master-Detail Menjadi `details[]`](#master-detail-menjadi-details)). RDF lama tanpa `fieldValidation` hanya mengonversi field header, disertai warning |
| Field layout | Tidak generate `fieldRows` (grid layout). Perlu ditambahkan manual di UDF setelah migrasi. `fieldRows` manual dipertahankan saat migrate ulang, dan kolom baru ditambahkan sebagai baris baru |
| Custom validation | Validasi UDF (mis. regex pattern) tidak di-generate. Hanya `required` yang dipindahkan |
| SQL parser | Parser memakai regex untuk pattern standar. Query dengan subquery, CASE, atau aggregate function mungkin tidak terurai sempurna |
| JOIN tanpa sibling RDF | Bila tabel JOIN tidak punya file RDF di folder yang sama, page terkait tidak di-generate; hanya referensi `select` field FK yang dipertahankan dengan warning |
| Kolom JOIN memakai nama kolom tabel utama | Pola seperti `s.status_name AS status`, dengan `status` sebagai kolom FK, tidak didukung. Field `status` tidak dibuat di form, disertai warning: `"<table>.<field>: datatablesQuery selects joined column '<ekspresi>' under the name of a main table column, so field '<field>' is not generated; select the main table column itself and give the joined column a different name"`. Aturan lengkapnya ada di [Aturan Kolom JOIN di `datatablesQuery`](../../../catalogs/rdf/validation-rules.md#aturan-kolom-join-di-datatablesquery) |
| `decimalPlaces` dari RDF lama (`constraints.precision`) | RDF hasil `payload generate` versi lama, yang belum mengisi `scale` dengan benar, memicu satu warning per page: `"<table>: decimalPlaces for field(s) <field list> was derived from constraints.precision because this RDF predates issue-91 (its scale was not populated); run 'npx restforge payload generate' again to refresh 'scale' and remove this fallback"`. Field header dan field detail digabung dalam satu warning yang sama |

## Workflow Lengkap

```
┌─────────────────┐  payload migrate  ┌─────────────────────┐  npx restforge-designer generate  ┌─────────────────┐
│ payload/        │ ────────────────→ │ frontend/payload/   │ ────────────────────────────→ │ ./build/        │
│ visitors.json   │                   │   app-config.json   │                               │ visitors-app/   │
│ (RDF, ber-JOIN) │                   │   pages\*.json      │                               │ (HTML/JS/CSS)   │
└─────────────────┘                   │   visitors-app.json │                               └─────────────────┘
                                       │   (aggregator/UDF)  │
                                       └─────────────────────┘
```

Langkah:

1. **Migrasi backend → frontend**:
   ```bat
   npx restforge payload migrate --name=visitors.json --project=visitors-app --output=..\frontend\payload --config=db-connection.env
   ```
2. **Edit manual** (opsional): tambahkan `fieldRows`, ubah label, sesuaikan icon, atau
   tambahkan `navigation.icon` langsung di fragmen `pages\` atau aggregator. Perubahan ini
   tetap ada saat langkah 1 diulang setelah RDF berubah, selama `--overwrite` tidak dipakai.
3. **Validasi** (menunjuk ke aggregator `<appCode>.json`, bukan fragmen `pages\`):
   ```bat
   npx restforge-designer validate --payload=..\frontend\payload\visitors-app.json
   ```
4. **Generate aplikasi frontend** (selalu menunjuk ke aggregator):
   ```bat
   npx restforge-designer generate --payload=..\frontend\payload\visitors-app.json --output=.\build\visitors-app --overwrite
   ```

## Hubungan dengan Binary Lain

| Tahap | Binary | Command |
|-------|--------|---------|
| RDF generate | `restforge` (backend) | `npx restforge payload generate ...` |
| RDF → UDF migrate | `restforge` (backend) | `npx restforge payload migrate ...` ← halaman ini |
| UDF generate ke web app | `restforge-designer` (frontend) | `npx restforge-designer generate --payload=<appCode>.json ...` |

---

**Lihat juga**: [`payload/`](./) · [`payload generate`](./generate.md) · [`payload validate`](./validate.md) · [`commands/`](../) · [`catalogs/udf/`](../../../catalogs/udf/) · [`catalogs/rdf/`](../../../catalogs/rdf/) · [`README`](../../../README.md)
