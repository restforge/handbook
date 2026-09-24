# Konvensi CLI

> Aturan umum yang berlaku untuk semua perintah RESTForge.

## Konvensi Flag-Only

Seluruh argument di RESTForge CLI menggunakan **flag eksplisit**. Pattern utama tetap `npx restforge <resource> <verb> [options]` di mana `<resource>` dan `<verb>` adalah positional wajib di posisi pertama dan kedua. Selain itu, seluruh argument lain dideklarasikan sebagai flag bernama (`--name=<VALUE>`).

### Aturan

1. **Resource dan verb wajib positional** — selalu di posisi pertama dan kedua setelah `restforge`, mis. `restforge schema validate`
2. **Seluruh argument lain wajib flag-only** — tidak ada positional argument tambahan setelah verb
   - Path filesystem: `--schema-path=<PATH>`, `--output=<PATH>`, `--config=<FILE>`
   - Identifier non-path: `--project=<NAME>`, `--table=<NAME>`, `--dialect=<TYPE>`
   - Boolean option: `--dry-run`, `--force` (bare flag, setara `--flag=true`)

### Contoh Aplikasi

| Command | Pattern | Catatan |
|---|---|---|
| `schema init --schema-path=<PATH>` | Flag-only | Path file scaffold target |
| `schema validate --schema-path=<PATH>` | Flag-only | Path folder schema (wajib) |
| `schema models --schema-path=<PATH>` | Flag-only | Path folder schema (wajib) |
| `schema generate-ddl --schema-path=<PATH> --dialect=<TYPE>` | Flag-only | Multiple flag |
| `schema migrate --schema-path=<PATH> --config=<FILE>` | Flag-only | Multiple flag dengan peran berbeda |
| `schema diff --schema-path=<PATH> --config=<FILE>` | Flag-only | Read-only drift detection |
| `schema apply --schema-path=<PATH> --config=<FILE>` | Flag-only | Apply incremental ALTER |
| `schema introspect --config=<FILE> --schema-path=<PATH>` | Flag-only | Multiple path role dengan disambiguasi flag |
| `payload generate --config=<FILE>` | Flag-only | Tidak ada path argument |

### Rationale

Konsistensi flag-only memberi keuntungan berikut:

- **Eksplisit dan jelas** — peran setiap argument langsung jelas dari nama flag, tanpa perlu mengingat urutan positional
- **Konsisten lintas command** — user tidak perlu memikirkan command mana yang pakai positional dan mana yang pakai flag
- **Ergonomis untuk scripting** — order tidak relevan, mudah untuk template generation di CI/CD
- **Mudah diperluas** — penambahan flag baru tidak memerlukan keputusan positional vs flag

### Catatan Historis

Sebelumnya beberapa schema command menerima `<path>` sebagai positional argument (mis. `schema validate ./schema`). Setelah refactor flag-only, sintaks positional dihapus sepenuhnya (hard-break, tanpa backward compatibility). Pemanggilan dengan sintaks lama akan menghasilkan error parser `Missing required flag: --schema-path`.

## Penamaan Lokasi SDF

Lokasi file/folder SDF dirujuk dengan flag `--schema-path` di seluruh command RESTForge:

- Command primitif resource `schema` (`init`, `validate`, `models`, `generate-ddl`, `migrate`, `diff`, `apply`, `template`): menggunakan `--schema-path` sebagai path ke file atau folder SDF.
- Command tingkat-aplikasi / orchestrator (`data pull`, `data push`, `fast-track`): juga menggunakan `--schema-path` untuk lokasi SDF.

Flag `--schema` (tanpa `-path`) TIDAK dipakai untuk lokasi SDF; flag tersebut khusus untuk **filter namespace database** pada `schema introspect`, `schema list`, serta `data pull` / `data push`.

## Global Options

Flag berikut berlaku pada level binary `restforge`:

| Flag | Keterangan |
|------|-----------|
| `--help` / `-h` | Tampilkan help message |
| `--version` / `-v` | Tampilkan versi package |
| `--archive-retain=<n>` | Jumlah folder run arsip yang disimpan di `.restforge/archive/` (default `5`, `0` berarti simpan semua). Lihat [Arsip File Lama](#arsip-file-lama) |

Help spesifik per command dapat diakses dengan menjalankan command tersebut diikuti `--help`, contoh: `npx restforge schema introspect --help`.

## Arsip File Lama

Command yang menimpa file di project menyimpan versi lama file tersebut ke folder `.restforge/archive/` di root project sebelum menulis versi baru. Direktori kerja karena itu hanya berisi file aktif.

Satu kali command menghasilkan satu folder run yang dinamai dengan stempel waktu `YYYYMMDD-HHMMSS`. Di dalam folder run, file disimpan pada path relatif aslinya dengan nama yang sama, sehingga isinya dapat langsung dibandingkan dengan file aktif memakai tool diff biasa. Contoh isi satu folder run:

```
.restforge/archive/20260924-101500/manifest.json
.restforge/archive/20260924-101500/payload/product.json
.restforge/archive/20260924-101500/payload/query/product-datatables.sql
.restforge/archive/20260924-101500/frontend/payload/pages/product.json
```

File `manifest.json` mencatat command pemicu, waktu mulai, dan daftar pasangan path asal serta path arsip. Nilai flag yang sensitif, seperti password, disamarkan di manifest.

### Cakupan

| Command | File yang diarsipkan |
|---|---|
| `payload generate` | RDF di `payload/` dan query SQL di `payload/query/`, bila isinya berubah |
| `payload sync` | RDF yang di-update, beserta query SQL bila `--expand-fk` dipakai |
| `payload migrate --overwrite` | UDF di folder output (`app-config.json`, file agregator project, dan `pages/*.json`), bila isinya berubah |
| `endpoint create --force` | File module dan model endpoint |
| `processor create --force` | File implementasi processor |
| `dashboard create --force` | File module dashboard |
| `schema introspect --force`, `schema template --generate --force` | File SDF yang ditimpa |
| `fast-track --overwrite` | Seluruh file yang ditimpa oleh setiap langkah, termasuk langkah frontend |
| `restforge-designer generate --overwrite`, `restforge-designer auth --overwrite` | File frontend yang ditimpa |

Satu command yang menjalankan beberapa langkah, misalnya `fast-track`, tetap menghasilkan satu folder run untuk seluruh langkahnya. Bila file yang sama ditimpa lebih dari sekali dalam satu command, yang tersimpan adalah versi sebelum command dijalankan.

### Memulihkan Satu Run

Isi folder run dapat disalin kembali ke root project untuk membatalkan perubahan satu command. Contoh di Windows CMD:

```bat
cd /d D:\projects\my-app
robocopy .restforge\archive\20260924-101500 . /E /XF manifest.json
```

Opsi `/XF manifest.json` mengecualikan manifest karena file itu bukan bagian dari project.

### Retensi

Secara default hanya lima folder run terbaru yang disimpan. Folder run yang lebih lama dihapus saat command berikutnya membuat folder run baru. Jumlah ini diatur lewat flag global `--archive-retain=<n>`, dan `--archive-retain=0` menyimpan seluruh folder run.

```bat
npx restforge endpoint create --project=my-app --name=orders --payload=orders.json --force --archive-retain=10
```

Folder `.restforge/archive/` sebaiknya dimasukkan ke `.gitignore` project agar arsip tidak ikut di-commit.

## Catatan Penggunaan

- **Resolusi `--config`**: jika `--config` tidak disediakan secara eksplisit, command yang menerima parameter ini (resource `payload`, `schema`, `query`) akan fallback ke default config dari `.restforge/defaults.json`. Saat fallback aktif, warning ditampilkan ke stderr agar pengguna mengetahui config mana yang dipakai. Parameter `--config` eksplisit selalu memiliki prioritas lebih tinggi daripada default.
- **Flag `--force`** bersifat **destruktif** dan harus digunakan dengan hati-hati karena file yang sudah ada tertimpa tanpa konfirmasi. Versi lama tetap tersimpan di `.restforge/archive/` (lihat [Arsip File Lama](#arsip-file-lama)).
- **Command read-only** yang aman dijalankan kapan saja: `validate`, `payload diff`, `schema list`, `schema describe`, `schema validate`, `schema models`, `schema generate-ddl` (tanpa `--output`), `query validate`, seluruh `catalog *`, serta `config get-default` dan `config list`. Command ini tidak melakukan modifikasi pada filesystem maupun database.
- **Exit code**: `0` untuk sukses, `≠ 0` untuk failure (validation error, file write error, license invalid, database connection error, dll.). Konsisten untuk integrasi scripting dan CI/CD.
- **Boolean flag format**: `--flag=true|false` (eksplisit) atau short form `--flag` (implicit true). Contoh: `--cluster` setara `--cluster=true`, sedangkan `--pretty=false` digunakan untuk menonaktifkan default boolean.
- **Instalasi lokal wajib**: seluruh command dijalankan dengan `npx restforge ...` dari dalam folder project agar binary ter-resolve dari instalasi lokal project. Instalasi global (`npm install -g @restforgejs/platform`) tidak didukung dan ditolak, baik saat instalasi maupun saat binary dari salinan global dijalankan; keduanya menghasilkan pesan error berisi langkah perbaikan. Jalur instalasi adalah lokal per project via `npx create-restforge-app` (project baru) atau `npm install @restforgejs/platform` (folder project yang sudah ada).
- **Subcommand wajib eksplisit**: invocation `restforge` tanpa subcommand (mis. `restforge --project=X`) akan menghasilkan error. Gunakan pattern resource-first yang sesuai, mis. `npx restforge serve --project=X`.
- **Debug logging**: set environment variable `DEBUG=1` untuk extra logging pada CLI command jika diperlukan troubleshooting.
- **Interactive vs non-interactive**: beberapa command seperti `key revoke` (tanpa `--file`) akan masuk mode interaktif. Pada lingkungan non-interaktif (CI/CD), sediakan seluruh flag yang diperlukan agar command tidak menggantung menunggu input.

## Penulisan Single-Line di Dokumentasi

Semua contoh command di handbook ditulis dalam **satu baris** (tidak memakai line continuation seperti `\`, `^`, atau `` ` ``).

Alasan: line continuation berbeda per shell:

| Shell | Karakter Line Continuation |
|-------|---------------------------|
| Bash, Zsh, Git Bash | `\` (backslash) |
| Windows CMD | `^` (caret) |
| PowerShell | `` ` `` (backtick) |

Dengan single-line, contoh command dapat di-copy-paste langsung ke shell apa pun tanpa modifikasi. Pengguna yang ingin memecah jadi multi-line untuk script lokal dapat menambahkan karakter line continuation sesuai shell pilihan.

---

**Lihat juga**: [`commands/`](./) · [`README`](../README.md)
