# Domain `codegen_*`

> 27 tool yang membungkus verb generator backend: `schema` (SDF), `payload` (RDF),
> `endpoint`, `processor`, `dashboard`, `kafka`, `test`, `catalog`, dan `query`.

Seluruh tool di domain ini menjalankan binary `restforge` di dalam `cwd`, sehingga
`@restforgejs/platform` harus terpasang lokal di folder tersebut dan license harus valid.
Gerbang sebelum domain ini dipakai adalah `setup_validate_config`.

Kolom "parameter" menandai parameter wajib dengan **tebal**. Parameter `cwd` wajib pada seluruh
tool dan tidak diulang di kolom tersebut. Parameter `config` yang berdefault
`db-connection.env` tetap mengikuti resolusi CLI, termasuk fallback ke default yang tercatat di
`.restforge/defaults.json`.

## Introspeksi Database Live

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `codegen_list_tables` | `schema list --config=<config> --format=json` | `config` (default `db-connection.env`), `schema`, `includeSystem` | `--format=json` dikirim tetap agar hasilnya terstruktur; bentuk output human-readable tidak tersedia lewat tool ini. |
| `codegen_describe_table` | `schema describe --config=<config> --table=<table>` | **`table`**, `config` (default `db-connection.env`), `includeForeignKeys`, `includeIndexes` | Menjelaskan kolom, primary key, foreign key, dan index satu tabel dari database live, bukan dari file SDF. |

## SDF (`schema`)

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `codegen_dbschema_init` | `schema init --schema-path=<path>` | **`schemaPath`** | Membuat struktur folder SDF beserta contoh awal. |
| `codegen_dbschema_validate` | `schema validate --schema-path=<path>` | **`schemaPath`** | Validasi statis file SDF tanpa menyentuh database. Verb ini hanya menerima `--schema-path`; tidak ada mode output JSON pada CLI, sehingga laporan diteruskan dalam bentuk teks. |
| `codegen_dbschema_models` | `schema models --schema-path=<path>` | **`schemaPath`** | Mendaftar model yang terbaca dari folder SDF. Berguna sebagai pendamping validasi saat menyusun SDF. |
| `codegen_dbschema_generate_ddl` | `schema generate-ddl --schema-path=<path> --dialect=<X>` | **`schemaPath`**, **`dialect`**, `output`, `drop` | Menghasilkan DDL untuk dialect yang diminta tanpa menjalankannya. Bila `output` diisi, DDL ditulis ke file. |
| `codegen_dbschema_migrate` | `schema migrate --schema-path=<path> --config=<file>` | **`schemaPath`**, `config` (default `db-connection.env`), `drop`, `dryRun`, `maxNameLength`, `autoCreateDb` | Menerapkan SDF ke database. Bersifat mutasi: `drop` menghapus objek dan `autoCreateDb` membuat database bila belum ada, keduanya hanya dikirim saat diminta. `dryRun` menampilkan rencana tanpa mengeksekusi. |
| `codegen_dbschema_introspect` | `schema introspect --config=<file> --schema-path=<path>` | **`schemaPath`**, `config` (default `db-connection.env`), `table`, `schema`, `allSchemas`, `force`, `dryRun` | Arah kebalikan dari migrate: struktur database dibaca lalu ditulis menjadi file SDF. `force` menimpa file SDF yang sudah ada, sehingga menjadi jalur destruktif terhadap definisi yang sedang dikerjakan. |
| `codegen_dbschema_diff` | `schema diff --schema-path=<path> --config=<file> --json` | **`schemaPath`**, `config` (default `db-connection.env`), `table` | `--json` dikirim tetap dan laporan drift di-parse menjadi struktur (versi, ringkasan, bagian per tabel). Read-only. |
| `codegen_dbschema_apply` | `schema apply --schema-path=<path> --config=<config>` | **`schemaPath`**, `config` (default `db-connection.env`), `table`, `dryRun` (default `false`), `allowDrop` (default `false`), `allowModify` (default `false`) | Menutup drift yang ditemukan `codegen_dbschema_diff`. Operasi merusak data dikunci di balik `allowDrop` dan `allowModify`; tanpa keduanya, statement yang berisiko dilewati. Output berupa laporan terstruktur untuk dibaca manusia, bukan JSON. |
| `codegen_dbschema_template` | `schema template [filter/mode]` | `domain`, `table`, `category`, `pattern`, `section`, `hasSdf`, `noSdf`, `show`, `example`, `lang`, `generate`, `schemaPath`, `force`, `listDomains`, `listCategories`, `listSections`, `stats`, `format` | Katalog template SDF referensi. Flag boolean dikirim hanya dalam bentuk bare flag saat bernilai `true`, sehingga nilai `false` tidak pernah diteruskan. Kombinasi `generate` dengan `schemaPath` menulis file SDF, dan `force` menimpanya; mode lain bersifat read-only. |

## RDF (`payload`)

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `codegen_generate_payload` | `payload generate --table=<table> --config=<config>` | **`table`**, `config` (default `db-connection.env`), `output`, `schemaPath`, `detail` | Menghasilkan RDF satu tabel dari struktur database. `detail` mengaktifkan bentuk master-detail: payload memperoleh blok `masterDetail`, file query detail, dan composite action. Tabel detail harus punya foreign key ke primary key tabel master. Blok `masterDetail` yang sudah ada di file target dipertahankan, tidak ditimpa. |
| `codegen_validate_payload` | `payload validate --config=<config>` | `config` (default `db-connection.env`), `table` | Validasi RDF terhadap catalog dan struktur database. Tanpa `table`, seluruh payload di folder diperiksa. |
| `codegen_diff_payload` | `payload diff --config=<config>` | `config` (default `db-connection.env`), `table` | Membandingkan RDF terhadap kolom database secara read-only, tanpa menulis perubahan. |
| `codegen_sync_payload` | `payload sync --config=<config>` | `config` (default `db-connection.env`), `table`, `expandFk`, `fkColumns` | Menuliskan kembali perubahan struktur database ke file RDF. Mode ekspansi FK bersifat opt-in dan **mewajibkan** `table`; panggilan `expandFk` tanpa `table` ditolak lebih dulu di sisi tool sebagai precondition. `fkColumns` berisi daftar `tabel.kolom` yang dipisah koma; bila dikosongkan, kolom tampilan per FK di-resolve otomatis dengan urutan name/nama, lalu code/kode, lalu primary key. |
| `codegen_migrate_payload` | `payload migrate --name=<name> --project=<project>` | **`name`**, **`project`**, `output`, `config`, `appName`, `appCode`, `plugin`, `port`, `overwrite` | Mengonversi RDF backend menjadi set UDF multi-file untuk frontend. Verb ini milik CLI backend meskipun keluarannya UDF, karena itu tool-nya berada di domain `codegen_*` dan bukan `designer_*`. Ini adalah jalur pertama pembuatan UDF; menulis UDF dari nol hanya wajar bila memang tidak ada RDF yang bisa dikonversi. |

## Generator Modul

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `codegen_create_endpoint` | `endpoint create --project=<p> --name=<endpoint> --payload=<payload>` | **`project`**, **`endpoint`**, **`payload`**, `database`, `createDemo`, `skipSqlValidation`, `noAuditMigration`, `skipSchemaCheck`, `config`, `verbose`, `force` (default `true`) | Destruktif pada jalur default: `force` berdefault `true` sehingga `--force=true` dikirim dan modul lama ditimpa, dengan versi sebelumnya diarsipkan CLI sebagai `*.archive.NNN`. Dengan `force=false`, flag tidak dikirim dan CLI menjalankan pemeriksaan konflik sendiri lalu berhenti tanpa menulis. Tipe database di-resolve CLI, bukan oleh tool: nilai `database` yang diisi selalu menang, bila dikosongkan CLI mendeteksi `DB_TYPE` dari config aktif, dan barulah jatuh ke `postgres`. Karena itu `database` sebaiknya dibiarkan kosong kecuali pengguna menyebut database secara eksplisit. Parameter `createDemo` dipetakan ke flag CLI `--create-examples`; nama lama dipertahankan demi kompatibilitas client. |
| `codegen_create_processor` | `processor create --project=<p> --name=<name> --payload=<payload>` | **`project`**, **`name`**, **`payload`**, `database`, `force`, `skipSqlValidation` | `force` tidak berdefault `true` di sini, jadi jalur default adalah jalur non-overwrite CLI. Bentuk `payload` mengikuti konvensi yang sama dengan endpoint: nama telanjang, boleh dengan atau tanpa `.json`, dan bentuk path ditolak. |
| `codegen_create_dashboard` | `dashboard create --project=<p> --name=<name> --payload=<payload> --database=<db>` | **`project`**, **`name`**, **`payload`**, `database`, `skipSqlValidation`, `force` (default `true`) | Nama dashboard wajib berawalan `dash-` dan menjadi segmen URL `POST /api/{project}/{name}/dashboard`. Berbeda dari endpoint, handler `dashboard create` **tidak** mendeteksi `DB_TYPE` dari config: tool selalu mengirim `--database`, dengan `postgres` sebagai nilai bila parameter dikosongkan. Untuk project non-postgres, `database` harus diisi eksplisit karena dialect ikut tertanam di SQL modul hasil generate. Nilai `payload` dipakai verbatim dan wajib memuat `.json`; CLI mencari di `<cwd>/payload/` lalu `<cwd>` tanpa pernah menambahkan ekstensi. Dengan `force=true` (default) CLI juga mendaftarkan ulang project di bawah nilai `database` panggilan ini, sehingga dialect yang tercatat sebelumnya tergantikan tanpa peringatan. |
| `codegen_validate_dashboard_payload` | `dashboard create --validate-only=true` | **`project`**, **`name`**, **`payload`**, `database`, `skipSqlValidation` | Read-only: tidak menulis file, tidak menyentuh database, tidak memperbarui registry project. Membutuhkan versi `@restforgejs/platform` yang sudah memuat flag `--validate-only` pada `dashboard create`; flag itu masuk ke source platform **setelah** rilis 5.5.5. Pada platform lama CLI menjawab `Unknown flag: --validate-only`, dan tool melaporkannya sebagai kebutuhan upgrade, bukan sebagai payload bermasalah. Argumen `project`, `name`, dan `database` tetap wajib karena argument parser CLI memeriksanya walau nilainya tidak dipakai. |
| `codegen_create_kafka_consumer` | `kafka consumer-create --project=<p> --name=<name> --payload=<payload>` | **`project`**, **`name`**, **`payload`**, `force` | Tool ini sengaja tidak melakukan pre-check keberadaan file payload; resolver CLI untuk verb ini menerima nama maupun path, dan kegagalan dilaporkan langsung dari CLI. |
| `codegen_generate_test` | `test generate --project=<p> --endpoint=<endpoint>` | **`project`**, **`endpoint`**, `port`, `init`, `force` | Menghasilkan integration test (Jest dan Supertest) untuk endpoint yang sudah ada. Endpoint tidak di-pre-check oleh tool. `init` menyiapkan konfigurasi test-data project dan berguna pada test pertama. Menjalankan test tetap urusan pengguna. |

## Catalog

Keempat tool berikut read-only dan menjadi rujukan struktur sebelum payload disusun atau diedit.

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `codegen_get_field_validation_catalog` | `catalog field-validation` | (hanya `cwd`) | Spesifikasi aturan `fieldValidation` pada RDF. |
| `codegen_get_query_declarative_catalog` | `catalog query-declarative` | (hanya `cwd`) | Spesifikasi blok query declarative pada RDF. |
| `codegen_get_dashboard_catalog` | `catalog dashboard` | (hanya `cwd`) | Spesifikasi struktur payload dashboard, termasuk aturan widget. |
| `codegen_get_dbschema_catalog` | `catalog dbschema [--section] [--name] [--kind]` | `section`, `name`, `kind` | Spesifikasi SDF. Ketiga filter mempersempit potongan catalog yang dikembalikan. |

## Validasi SQL

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `codegen_validate_sql` | `query validate --config=<config> --sql=<sql> --pretty=false` | **`sql`**, `config` (default `db-connection.env`) | Verb dipanggil dalam bentuk dua token berspasi (`query validate`); bentuk berkolon tidak dikenali CLI sama sekali dan berujung `Unknown command`. `--pretty=false` dikirim tetap agar keluaran JSON ringkas, lalu diformat ulang oleh tool. Validasi memakai `EXPLAIN` terhadap database live dan hanya menerima `SELECT` atau CTE, sehingga tidak mengeksekusi perubahan data. Verb ini tersedia pada platform 5.5.5; versi minimum yang tepat belum ditetapkan. |
