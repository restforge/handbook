# Domain `setup_*`

> 13 tool untuk menyiapkan folder project, memasang `@restforgejs/platform`, mengelola file
> `config/*.env`, dan menentukan config default per working directory.

Domain ini adalah pintu masuk pipeline. Tool di domain lain mengandalkan hasilnya: package
yang terpasang lokal, file config yang terisi, dan koneksi yang sudah lolos validasi.

Kolom "parameter" menandai parameter wajib dengan **tebal**; sisanya opsional, dengan nilai
default disebutkan bila ada. Seluruh tool selain `setup_create_folder` menerima `cwd` sebagai
path absolut folder project.

## Penyiapan Project

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `setup_create_folder` | Operasi file langsung | `folderName` (default `backend-server`), `parentCwd`, `force` (default `false`) | Membuat folder secara rekursif. Folder yang sudah ada dilaporkan sebagai precondition, bukan error; `force` meneruskan pembuatan karena pembuatan rekursif bersifat idempoten. Tool ini adalah jalur granular; untuk scaffold satu langkah, `npx create-restforge-app` tetap jalur yang disarankan. |
| `setup_install_package` | `npm install @restforgejs/platform@<version>` | **`cwd`**, `version` (default `beta`) | Satu-satunya tool yang memanggil `npm`, bukan `restforge`. Instalasi selalu lokal ke `cwd`; tidak ada jalur global. Default `beta` dipakai karena platform masih pra-rilis publik; isi `latest` atau versi spesifik bila diperlukan. |
| `setup_init_config` | `restforge init` | **`cwd`** | CLI mendeteksi bahwa dirinya dipanggil di luar terminal interaktif, sehingga dialog dilewati dan template default langsung ditulis ke `config/db-connection.env`. Nilainya masih placeholder dan perlu diisi lewat `setup_write_env`. |

## Isi dan Baca Config

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `setup_write_env` | Operasi file langsung | **`cwd`**, **`license`**, **`dbType`**, **`dbHost`**, **`dbPort`**, **`dbUser`**, **`dbPassword`**, **`dbName`**, `serverAddress` (default `127.0.0.1`), `serverPort` (default `3000`) | Menulis blok license, server, dan database sekaligus dengan strategi merge parsial: entri yang cocok diperbarui di tempat, kunci yang belum ada ditambahkan di bawah, sedangkan komentar, baris kosong, dan parameter lain dipertahankan apa adanya. `license` divalidasi terhadap format `XXXX-XXXX-XXXX-XXXX`, `dbType` terbatas pada `postgresql`, `mysql`, `oracle`, `sqlite`. Target file tetap `config/db-connection.env`. |
| `setup_update_env` | Operasi file langsung | **`cwd`**, **`fields`**, `configFile` (default `db-connection.env`) | Update parsial untuk sekumpulan pasangan kunci-nilai bebas, termasuk kunci yang belum ada di template. Nilai boleh string, angka, atau boolean; nilai yang memuat spasi, `=`, atau `#` dikutip secara otomatis. Komentar inline pada baris yang diperbarui ikut dipertahankan. |
| `setup_read_env` | Operasi file langsung | **`cwd`**, `configFile` (default `db-connection.env`), `unmask` (default `false`) | Read-only dan aman dipanggil berulang. `LICENSE`, `DB_PASSWORD`, `REDIS_PASSWORD`, dan `KAFKA_SASL_PASSWORD` ditutup secara default; `unmask` membuka nilai aslinya. File yang belum ada dijawab sebagai precondition. |
| `setup_get_init_template` | `config template` | **`cwd`** | Mengembalikan isi mentah template `db-connection.env` tanpa menulis file apa pun. |
| `setup_get_config_schema` | `config schema` | **`cwd`** | Mengembalikan schema JSON seluruh parameter yang dikenali `db-connection.env`, termasuk section opsional seperti Redis, Kafka, dan Live Sync. Dipakai sebagai rujukan sebelum mengisi nilai lewat `setup_update_env`. |

## Validasi

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `setup_validate_config` | `restforge validate --config=<configFile> [--auto-create-db]` | **`cwd`**, `configFile` (default `db-connection.env`), `autoCreateDb` | Satu-satunya tool `setup_*` yang benar-benar membuka koneksi: license diverifikasi ke license server, lalu database, Redis, dan Kafka dihubungi sesuai isi config. Karena itu tool ini menjadi gerbang sebelum tool `codegen_*` dipanggil, dan bukan alat inspeksi awal. `autoCreateDb` membuat database bila belum ada, sehingga bersifat mutasi dan hanya dikirim saat diminta. |

## Config Default per Working Directory

Kelima tool berikut mengelola catatan config default di `.restforge/defaults.json`. Default itu
yang dipakai tool lain ketika parameter `config` dikosongkan, sehingga nama file config tidak
perlu diulang pada tiap panggilan.

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `setup_list_configs` | `config list` | **`cwd`** | Mendaftar file `.env` yang tersedia di folder `config/`. Output berbentuk JSON. |
| `setup_set_default_config` | `config set-default --config=<config>` | **`cwd`**, **`config`** | Mencatat satu config sebagai default. Nilai `config` ditelusuri berurutan: di `cwd`, lalu di folder `config/`, lalu dengan penambahan ekstensi `.env`. |
| `setup_get_default_config` | `config get-default` | **`cwd`** | Menampilkan config yang sedang tercatat sebagai default. Output berbentuk JSON. |
| `setup_clear_default_config` | `config clear-default` | **`cwd`** | Menghapus catatan default. Setelahnya setiap panggilan harus menyebut config-nya secara eksplisit. |
