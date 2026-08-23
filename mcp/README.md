# MCP Server

> Spec publik surface tool `@restforgejs/mcp-server`, yaitu MCP server yang membuka command RESTForge kepada AI agent (Claude Code, Claude Desktop, Cursor, dan client MCP lain).

MCP server ini adalah orchestrator tipis. Hampir seluruh tool-nya membungkus satu verb CLI
`restforge` atau `restforge-designer` yang sudah didokumentasikan di [`commands/`](../commands/),
lalu menambahkan pre-check dan format respons yang mudah dicerna agent. Sebagian kecil tool
tidak memanggil CLI sama sekali dan hanya membaca atau menulis file di folder project.

Halaman-halaman di folder ini adalah rujukan tunggal untuk pertanyaan "tool apa saja yang ada,
verb mana yang dibungkusnya, dan di mana perilakunya berbeda dari CLI".

## Instalasi dan Registrasi

MCP server tidak dipasang secara global. Registrasi dilakukan per client MCP, dan perintahnya
dijalankan lewat `npx` sehingga versi terbaru selalu ter-resolve saat client menyalakan server.

Jalur yang disarankan adalah installer skill, karena installer menaruh skill RESTForge dan
entri MCP sekaligus:

```bash
npx create-restforge-skills
```

Installer menulis entri berikut ke config MCP client (merge, bukan overwrite):

```json
{
  "mcpServers": {
    "restforge": {
      "command": "npx",
      "args": ["-y", "@restforgejs/mcp-server"]
    }
  }
}
```

Entri yang sama dapat ditulis manual bila registrasi ingin dikelola sendiri. Package ini juga
mendaftarkan binary `restforge-mcp`; binary tersebut hanya relevan bila package sudah tersedia
di lingkungan yang menjalankan client, dan bukan jalur pemasangan yang disarankan.

## Prasyarat

| Prasyarat | Keterangan |
|---|---|
| Node.js >= 18 | Runtime MCP server itu sendiri |
| `@restforgejs/platform` di `node_modules` project target | Hampir semua tool menjalankan `npx restforge ...` di dalam folder project, jadi package harus terpasang lokal di folder tersebut. Instalasi global tidak didukung. |
| License RESTForge | Diperlukan oleh tool yang menyentuh runtime platform: seluruh domain `codegen_*`, seluruh domain `runtime_*`, `data_*`, dan `setup_validate_config`. Tanpa license, tool tersebut mengembalikan error autentikasi dari CLI. |

Domain `designer_*` berjalan tanpa license karena mekanisme license sudah dilepas dari binary
`restforge-designer`. Binary tersebut tetap ikut terdistribusi di dalam `@restforgejs/platform`,
sehingga prasyarat "package terpasang lokal" tetap berlaku.

Package `@restforgejs/mcp-server` sendiri berlisensi MIT dan bebas dipasang maupun diperiksa.
Yang berlisensi komersial adalah platform yang diorkestrasinya.

## Domain Tool

Total **69 tool**, dikelompokkan ke sembilan domain berdasarkan prefix nama.

| Domain | Jumlah | Isi | Halaman |
|---|---|---|---|
| `setup_*` | 13 | Folder project, instalasi package, config `.env`, default config, validasi koneksi | [`setup.md`](./setup.md) |
| `codegen_*` | 27 | SDF (`schema`), RDF (`payload`), generator endpoint/processor/dashboard/kafka/test, catalog, query validate | [`codegen.md`](./codegen.md) |
| `designer_*` | 11 | Generator frontend: init project, plugin, catalog UDF, validate/preview/generate, auth overlay | [`designer.md`](./designer.md) |
| `runtime_*` | 7 | Deteksi project dan config, preflight, status server, penulisan launcher server dan consumer | [`runtime.md`](./runtime.md) |
| `data_*` | 2 | Export dan import baris data tabel lintas dialect | [`data.md`](./data.md) |
| `key_*` | 3 | Manajemen API key project | [`key.md`](./key.md) |
| `project_*` | 4 | Listing, penghapusan, ekstensi auth backend, generate SDK client | [`project.md`](./project.md) |
| `license_*` | 1 | Pembacaan aktivasi license mesin | [`license.md`](./license.md) |
| `health_*` | 1 | Smoke test transport MCP | [`health.md`](./health.md) |

Nama tool bersifat kontrak publik. Menambah atau menghapus tool tanpa memperbarui halaman
domain terkait terhitung drift, karena audit dua arah ("setiap tool punya spec, setiap spec
punya tool") dijalankan terhadap tabel di halaman-halaman tersebut.

## Pola Umum

Beberapa perilaku berlaku lintas domain dan tidak diulang pada tiap baris tabel.

### Parameter `cwd`

Hampir seluruh tool meminta `cwd`, yaitu path absolut folder project tempat perintah dijalankan.
Nilai ini menentukan instalasi `@restforgejs/platform` mana yang dipakai, config mana yang
ter-resolve, dan ke mana file hasil generate ditulis. Pengecualian: `health_ping` tidak punya
`cwd` sama sekali, `setup_create_folder` memakai `parentCwd`, dan `designer_get_udf_catalog`
memperlakukan `cwd` sebagai opsional karena catalog-nya konstanta statis.

### Precondition bukan error

Keadaan yang bisa diperbaiki pemanggil, misalnya package belum terpasang, file config belum ada,
atau file payload belum dibuat, dijawab sebagai respons biasa berisi penjelasan langkah
berikutnya, bukan sebagai error. Kegagalan CLI yang sesungguhnya barulah ditandai sebagai error.
Konsekuensinya, agent diarahkan menawarkan langkah setup alih-alih melaporkan kerusakan.

### Flag hardcode pada tool destruktif

Beberapa tool mengirim flag tetap yang tidak dapat dimatikan lewat parameter, karena tanpa flag
tersebut CLI akan menunggu jawaban prompt interaktif yang tidak mungkin dijawab dari MCP.

| Tool | Flag hardcode | Akibat |
|---|---|---|
| `project_delete` | `--yes` | Project dihapus langsung tanpa konfirmasi CLI |
| `key_revoke` | `--yes` | API key dicabut langsung tanpa konfirmasi CLI |
| `designer_auth_remove` | `--force` | Overlay auth dihapus langsung tanpa konfirmasi CLI |
| `codegen_create_endpoint`, `codegen_create_dashboard` | `--force=true` selama parameter `force` dibiarkan pada default `true` | File modul lama ditimpa; CLI mengarsipkan versi sebelumnya sebagai `*.archive.NNN` |

Karena konfirmasi CLI hilang, konfirmasi harus dilakukan agent kepada pengguna sebelum tool
dipanggil. Tool yang lain mengirim flag tetap hanya untuk urusan format output (`--json`,
`--format=json`, `--pretty=false`) dan tidak mengubah efek samping.

### Masking field sensitif

`setup_read_env` menutup nilai `LICENSE`, `DB_PASSWORD`, `REDIS_PASSWORD`, dan
`KAFKA_SASL_PASSWORD` secara default; parameter `unmask` membuka nilai aslinya.
`key_list` menampilkan API key dalam bentuk tertutup dan hanya membuka nilai penuh bila
`showFull` diaktifkan. Keduanya dirancang agar nilai rahasia tidak ikut masuk ke transcript
percakapan tanpa permintaan eksplisit.

### Pola dua langkah untuk launcher

MCP server tidak pernah menjalankan runtime server maupun Kafka consumer. Proses yang
dinyalakan dari sesi AI akan ikut mati saat sesi ditutup, sehingga kepemilikan proses harus
tetap berada di terminal pengguna.

Yang dilakukan tool adalah menulis file launcher, lalu pengguna yang mengeksekusinya:

```
runtime_detect_project → runtime_detect_config → runtime_validate_preflight
  → runtime_check_launcher_exists → runtime_generate_launcher → pengguna menjalankan skrip
```

`runtime_check_status` boleh dipanggil kapan saja karena hanya membaca keadaan dan tidak
mengubah siklus hidup proses. Pola yang sama berlaku untuk consumer lewat
`runtime_generate_consumer_launcher`.

### Konvensi nilai `payload`

Nilai parameter `payload` diteruskan apa adanya ke CLI, dan tiap verb generator me-resolve-nya
dengan aturan sendiri. Perbedaan ini nyata dan sering menjadi sumber kegagalan saat dua
generator dipanggil dalam satu sesi.

| Tool | Bentuk yang diterima |
|---|---|
| `codegen_create_endpoint`, `codegen_create_processor` | Nama telanjang, dengan atau tanpa `.json`; diubah menjadi huruf kecil oleh CLI; bentuk path ditolak |
| `codegen_create_dashboard`, `codegen_validate_dashboard_payload` | Nama atau path relatif yang **wajib** memuat `.json`; dipakai verbatim, tidak ada ekstensi yang ditambahkan |
| `codegen_create_kafka_consumer` | Nama atau path |

### Resolusi config lintas panggilan

Sebagian besar tool backend menerima parameter `config` opsional. Bila dikosongkan, CLI jatuh ke
default yang tercatat per working directory di `.restforge/defaults.json`. Default itu dikelola
lewat `setup_list_configs`, `setup_set_default_config`, `setup_get_default_config`, dan
`setup_clear_default_config`, sehingga nama file config tidak perlu diulang di setiap panggilan.

## Verb yang Sengaja Tidak Di-wrap

Tiga kelompok command tetap berada di luar surface tool, dengan alasan berbeda. Agent
menjelaskan dan menyerahkan perintahnya kepada pengguna, bukan menjalankannya sendiri.

| Command | Alasan |
|---|---|
| `restforge fast-track` | Alurnya interaktif (input license dan parameter database, menu pilihan scope, konfirmasi akhir), dan MCP server tidak dapat menjawab prompt. Pipeline yang sama dapat direproduksi langkah demi langkah lewat tool `codegen_*` dan `designer_*`. |
| `restforge serve` beserta start dan stop runtime | Mengikuti pola dua langkah launcher: proses yang dinyalakan dari sesi AI mati bersama sesi tersebut, sehingga tool hanya menulis skrip dan pengguna yang menjalankannya. |
| `restforge license deactivate` | Efeknya melampaui folder project karena melepas seat aktivasi secara terpusat di license server, dan tidak dapat dibatalkan dari sisi ini. Pemeriksaan keadaan aktivasi tetap tersedia lewat `license_info`. |

## Rujukan Terkait

- [`commands/restforge-backend/`](../commands/restforge-backend/README.md) untuk detail flag tiap verb CLI backend
- [`commands/restforge-frontend/`](../commands/restforge-frontend/README.md) untuk detail flag tiap verb `restforge-designer`
- [`catalogs/`](../catalogs/) untuk struktur SDF, RDF, dan UDF yang menjadi input tool generator
