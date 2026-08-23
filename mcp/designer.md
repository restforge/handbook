# Domain `designer_*`

> 11 tool yang membungkus binary frontend `restforge-designer`: scaffold project, plugin,
> catalog UDF, validasi dan preview, generate, serta overlay auth.

Binary `restforge-designer` terpisah dari binary backend `restforge`, walau keduanya
terdistribusi di dalam package `@restforgejs/platform` yang sama. Keduanya tidak saling
menggantikan: permintaan SDF atau RDF tidak boleh diarahkan ke tool `designer_*`, dan
permintaan UDF tidak boleh diarahkan ke tool `codegen_*`.

Domain ini berjalan tanpa license karena mekanisme license sudah dilepas dari binary Designer.
Yang tetap diperlukan hanyalah `@restforgejs/platform` terpasang lokal di `cwd`; bila tidak ada,
tool menjawabnya sebagai precondition, bukan kegagalan.

Kolom "parameter" menandai parameter wajib dengan **tebal**. Parameter `cwd` wajib pada seluruh
tool kecuali `designer_get_udf_catalog`, dan tidak diulang di kolom tersebut.

## Alur Baku UDF

Jalur pertama pembuatan UDF adalah konversi dari RDF lewat `codegen_migrate_payload` di domain
`codegen_*`, bukan penulisan manual. Setelah UDF ada, rangkaiannya:

```
codegen_migrate_payload → designer_get_udf_catalog → designer_validate_payload
  → designer_preview_files → designer_generate
```

Tiga tool terakhir diarahkan ke file UDF aggregator, bukan ke fragment per halaman.

## Project dan Generate

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `designer_init_project` | `restforge-designer init --plugin=<plugin> --output=<output>` | **`plugin`**, **`output`**, `appName`, `appCode`, `apiBaseUrl`, `authAppCode`, `authApiUrl`, `port`, `idleTimeout`, `noAuth`, `overwrite`, `pluginsDir` | Membuat struktur project frontend beserta UDF awal (`payload.json`) dan asset plugin. `noAuth` dan `overwrite` dikirim sebagai bare flag hanya saat bernilai `true`. |
| `designer_generate` | `restforge-designer generate --payload=<payload> --output=<output>` | **`payload`**, **`output`**, `plugin`, `overwrite`, `scope`, `page`, `skipShared`, `pluginsDir` | Menghasilkan file frontend dari UDF. `scope` dan `page` mempersempit target generate, `skipShared` melewati asset bersama. Tanpa `overwrite`, file yang sudah ada tidak ditimpa. |
| `designer_validate_payload` | `restforge-designer validate --payload=<payload>` | **`payload`**, `pluginsDir` | Read-only. Dipakai sebelum preview dan generate untuk memunculkan kesalahan struktur lebih murah. |
| `designer_preview_files` | `restforge-designer preview --payload=<payload>` | **`payload`**, `pluginsDir` | Menampilkan daftar file yang akan dihasilkan tanpa menulis apa pun. |

## Catalog dan Plugin

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `designer_get_udf_catalog` | `restforge-designer catalog [--section=<X>]` | `cwd`, `section` | Rujukan resmi struktur UDF: tipe field yang sah, field `appConfig` yang wajib, enum, batasan, serta opsi widget, chart, dan data source dashboard. Catalog bersifat konstanta, sehingga `cwd` opsional dan tidak memengaruhi hasil. `section` menerima `fields`, `navigation`, `app-config`, `dashboard`, atau `all`; bila dikosongkan, flag tidak dikirim dan binary memakai default `all`. |
| `designer_list_plugins` | `restforge-designer plugins list` | `pluginsDir` | Mendaftar plugin yang tersedia. |
| `designer_inspect_plugin` | `restforge-designer plugins inspect --plugin=<plugin>` | **`plugin`**, `pluginsDir` | Menampilkan identitas satu plugin. Bukan sumber aturan penulisan UDF; untuk itu gunakan `designer_get_udf_catalog`. |
| `designer_scaffold_plugin` | `restforge-designer plugins scaffold --id=<id> --output=<output>` | **`id`**, **`output`**, `pluginsDir` | Membuat kerangka plugin baru. |

## Auth Frontend

Ketiga mode auth saling eksklusif; CLI hanya menerima tepat satu mode per pemanggilan.

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `designer_auth_create` | `restforge-designer auth --create --project=<project>` | **`project`**, `frontendPath`, `apiBaseUrl`, `overwrite` | Memasang overlay `rfx_auth` mandiri (`login.html`, `signup.html`, `js/rfx_auth.js`) dan menyuntikkan guard ke halaman yang ada. Pilihan wajar ketika sebuah app hanya butuh UI login dan signup. |
| `designer_auth_attach` | `restforge-designer auth --attach --project=<project>` | **`project`**, `frontendPath`, `apiBaseUrl`, `overwrite` | Memasang scaffold auth penuh pada project yang halamannya sudah ter-generate, tanpa menyentuh file halaman: kontrak `window.Auth` selalu terpasang, ditambah artefak login plugin bila payload project mengaktifkan auth pada `vanilla-js-auth` atau `vanilla-js-custom`. Dipakai saat `designer_generate` memperingatkan artefak auth belum ada, atau saat auth perlu dinyalakan belakangan. Flag `--force` tidak diekspos karena pada mode ini flag tersebut diabaikan CLI. |
| `designer_auth_remove` | `restforge-designer auth --remove --project=<project> --force` | **`project`**, `frontendPath` | Mencopot overlay auth. `--force` di-hardcode agar prompt konfirmasi CLI dilewati, sehingga penghapusan berlangsung seketika. Konfirmasi kepada pengguna harus dilakukan sebelum tool dipanggil. |
