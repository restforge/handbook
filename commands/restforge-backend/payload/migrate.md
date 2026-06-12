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

## Flag Wajib (Required)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--name <STRING>` | - | Nama file payload backend (mis. `visitors.json`). Path resolusi: relatif terhadap `cwd` atau `cwd/payload/` |
| `--project <STRING>` | - | Nama project (kebab-case). Dipakai sebagai segment path di `apiBaseUrl`: `http://{host}:{port}/api/{project}` |

## Flag Opsional (Optional)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--output <STRING>` | `frontend/payload/` | Output directory tempat file split ditulis. Jika nilai diakhiri `.json`, direktori induknya dipakai sebagai output directory (file tunggal tidak lagi dihasilkan) |
| `--config <STRING>` | `null` | Database config file (`.env`). Dibaca untuk ambil `SERVER_ADDRESS` dan `SERVER_PORT` (backend port). Fallback ke default config (`restforge config set-default`) jika tidak diisi |
| `--app-name <STRING>` | diturunkan dari `--project` (Title Case) | Nama aplikasi yang dipakai di `appConfig.appName` UDF output. Jika tidak diisi, di-derive dari `--project` dengan format Title Case, mis. `visitors-app` menjadi `"Visitors App"` |
| `--app-code <STRING>` | mengikuti `--project` | Kode aplikasi kebab-case yang dipakai di `appConfig.appCode` UDF output. Dipakai juga sebagai nama file aggregator (`<appCode>.json`) |
| `--plugin <STRING>` | `"vanilla-js-basic"` | Plugin ID Designer yang ditulis di `appConfig.plugin` UDF output |
| `--port <NUMBER>` | `8000` | Port aplikasi frontend yang ditulis ke `appConfig.port`. Independen dari backend port yang dipakai di `apiBaseUrl` |
| `--overwrite` | `false` | Timpa file output jika sudah ada. Gate berlaku untuk setiap file split yang sudah ada di output directory |

## Contoh (Examples)

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

### Migrasi dengan Overwrite Output Existing

```bat
npx restforge payload migrate --name=visitors.json --project=visitors-app --overwrite
```

Tanpa `--overwrite`, command gagal jika salah satu file split (mis. `app-config.json`
atau aggregator) sudah ada di output directory.

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

## Output Split Multi-File

Migrate menulis beberapa file ke dalam output directory, bukan satu file UDF tunggal.
Struktur yang dihasilkan:

| File | Isi |
|------|-----|
| `app-config.json` | Blok `appConfig` bersama: `appName`, `appCode`, `plugin`, `apiBaseUrl`, `port` |
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
    "port": 8000
  }
}
```

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

## Aturan Konversi RDF → UDF

Migrator menginferensi struktur UDF dari struktur RDF berdasarkan aturan deterministic.
Detail mapping field type, exclude rules, dan inferensi label ada di [`catalogs/udf/`](../../../catalogs/udf/).

Ringkasan:

| Aspek | Behavior |
|-------|----------|
| Tipe field | Auto-detect dari `fieldValidation` RDF (boolean → checkbox, number → number, string enum → select static, `*_date` → date, dst.) |
| FK + JOIN → select | Field FK yang menjadi local column sebuah JOIN dikonversi menjadi dropdown `select` dengan `dataSource` API |
| JOIN discovery | RDF utama ber-JOIN memuat RDF tabel terkait dari sibling folder dan mengonversinya menjadi page tambahan (1 RDF utama → multi page) |
| Field di-exclude | Primary key (`constraints.primaryKey: true`), audit columns (`created_at`, `created_by`, `updated_at`, `updated_by`), display column milik tabel JOIN |
| Properti page | `tableName` → `pageId` (kebab-case), `pageTitle` (Title Case), `apiPath` (sama dengan `pageId`) |
| Fitur otomatis | `datatablesWhere` memuat kolom searchable → `enableSearch: true`. Field `is_active`/`status` boolean → `enableStatusFilter: true` |
| Label dan placeholder | Auto-generate dari nama field |

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

## Batasan (Limitations)

| Aspek | Behavior |
|-------|----------|
| Master-detail (`masterDetail`) | RDF dengan blok `masterDetail` hanya dikonversi bagian header (master). Detail table belum didukung dan menghasilkan warning. Ini berbeda dari JOIN auto-discovery yang justru menghasilkan page master tersendiri |
| Field layout | Tidak generate `fieldRows` (grid layout). Perlu ditambahkan manual di UDF setelah migrasi |
| Custom validation | Validasi UDF (mis. regex pattern) tidak di-generate. Hanya `required` yang dipindahkan |
| SQL parser | Parser memakai regex untuk pattern standar. Query dengan subquery, CASE, atau aggregate function mungkin tidak terurai sempurna |
| JOIN tanpa sibling RDF | Bila tabel JOIN tidak punya file RDF di folder yang sama, page terkait tidak di-generate; hanya referensi `select` field FK yang dipertahankan dengan warning |

## Workflow Lengkap

```
┌─────────────────┐  payload migrate  ┌─────────────────────┐  restforge-designer generate  ┌─────────────────┐
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
   npx restforge payload migrate --name=visitors.json --project=visitors-app --output=..\frontend\payload --config=db-connection.env --overwrite
   ```
2. **Edit manual** (opsional): tambahkan `fieldRows`, ubah label, sesuaikan icon, atau
   tambahkan `navigation.icon` langsung di fragmen `pages\` atau aggregator.
3. **Validasi** (menunjuk ke aggregator `<appCode>.json`, bukan fragmen `pages\`):
   ```bat
   restforge-designer validate --payload=..\frontend\payload\visitors-app.json
   ```
4. **Generate aplikasi frontend** (selalu menunjuk ke aggregator):
   ```bat
   restforge-designer generate --payload=..\frontend\payload\visitors-app.json --output=.\build\visitors-app --overwrite
   ```

## Hubungan dengan Binary Lain

| Tahap | Binary | Command |
|-------|--------|---------|
| RDF generate | `restforge` (backend) | `npx restforge payload generate ...` |
| RDF → UDF migrate | `restforge` (backend) | `npx restforge payload migrate ...` ← halaman ini |
| UDF generate ke web app | `restforge-designer` (frontend) | `restforge-designer generate --payload=<appCode>.json ...` |

## Catatan Migrasi dari Command Lama

Sebelumnya migrasi RDF → UDF dilakukan via `restforge-designer migrate` (binary frontend). Command tersebut sudah **dihapus** dan digantikan oleh `npx restforge payload migrate` (binary backend). Alasan: konversi RDF → UDF memerlukan akses ke config database backend (mis. `SERVER_PORT`, `SERVER_ADDRESS`) yang lebih natural ditangani di sisi backend.

Perubahan flag dari command lama:

| Lama (`restforge-designer migrate`) | Baru (`npx restforge payload migrate`) |
|------------------------------------|--------------------------------------|
| `--input <FILE>` | `--name <STRING>` (resolved dari `cwd` atau `cwd/payload/`) |
| `--output <FILE>` | `--output <STRING>` (output directory; suffix `.json` diturunkan ke direktori induk) |
| `--api-base-url <URL>` | `--config <FILE>` + `--project <STRING>` (URL dibentuk otomatis) |
| `--app-name <STRING>` | `--app-name <STRING>` (default di-derive dari `--project` dalam Title Case) |
| `--app-code <STRING>` | `--app-code <STRING>` (default mengikuti `--project`) |
| `--plugin <STRING>` | `--plugin <STRING>` (sama) |
| `--port <PORT>` | `--port <NUMBER>` (frontend port, `appConfig.port`, default `8000`). Backend port dibaca dari `--config` (`SERVER_PORT`) |
| `--overwrite` | `--overwrite` (sama) |

---

**Lihat juga**: [`payload/`](./) · [`payload generate`](./generate.md) · [`payload validate`](./validate.md) · [`commands/`](../) · [`catalogs/udf/`](../../../catalogs/udf/) · [`catalogs/rdf/`](../../../catalogs/rdf/) · [`README`](../../../README.md)
