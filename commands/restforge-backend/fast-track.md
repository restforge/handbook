# `fast-track` (global verb)

> Generate REST API dan frontend app dari SDF secara cepat dalam satu alur dialog interaktif, lengkap dengan opsi menjalankan runtime server. Command ini mengorkestrasi command `restforge` yang sudah ada, bukan menambah logic generate baru.

Verb `fast-track` adalah verb global yang tidak terikat pada resource tertentu, sehingga pemanggilannya cukup `npx restforge fast-track` tanpa prefix resource.

## Pattern

```
npx restforge fast-track --project=<STRING> --schema-path=<STRING> [options]
```

## Flag Wajib (Required)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--project <STRING>` | - | Nama project/aplikasi (mis. `visitors-app`) |
| `--schema-path <STRING>` | - | Folder berisi SDF aplikasi (relatif terhadap cwd) |

## Flag Opsional (Optional)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--config <STRING>` | `db-connection.env` | Nama file env target di folder `config/` (ditulis dari input dialog) |
| `--license <STRING>` | - | License key (`XXXX-XXXX-XXXX-XXXX`). Bila diisi, dipakai sebagai default prompt LICENSE |
| `--overwrite` | `false` | Mode destruktif: drop table dan regenerate (perlu konfirmasi `y/N`) |

## Alur Dialog Interaktif (Interactive Flow)

Command berjalan interaktif melalui lima tahap berurutan:

1. **Input konfigurasi** — prompt LICENSE dan parameter database (lihat [Input Konfigurasi](#input-konfigurasi-configuration-input)). Tekan Enter untuk memakai nilai default.
2. **Pemilihan scope** — menu pilihan apa yang akan di-generate (lihat [Pilihan Scope](#pilihan-scope-scope-selection)).
3. **Preflight `restforge-designer`** — hanya dijalankan bila scope mencakup frontend (lihat [Dependency restforge-designer](#dependency-restforge-designer)).
4. **Preview dan konfirmasi** — ringkasan project, scope, database, dan mode ditampilkan sebelum eksekusi. Mode default minta konfirmasi `Y/n`, mode `--overwrite` minta `y/N`.
5. **Eksekusi pipeline dan opsi service** — pipeline dijalankan sesuai scope, lalu command menawarkan menjalankan runtime server dan frontend app di window baru.

## Input Konfigurasi (Configuration Input)

| Prompt | Default | Keterangan |
|--------|---------|-----------|
| `LICENSE` | sandbox built-in | License key runtime dan designer. Untuk production, isi via `--license` atau input manual |
| `SERVER_ADDRESS` | `127.0.0.1` | Alamat bind REST API server |
| `REST_API_PORT` | `3000` | Port REST API (disimpan ke env sebagai `SERVER_PORT`) |
| `WEB_SERVER_PORT` | `8000` | Port frontend app (dipakai oleh `npx serve`) |
| `DB_TYPE` | `postgresql` | Pilihan: `postgresql`, `mysql`, `oracle`, `sqlite` |
| `DB_HOST` / `DB_PORT` / `DB_USER` / `DB_PASSWORD` / `DB_NAME` | menyesuaikan `DB_TYPE` | Kredensial database (non-sqlite). Default port/user mengikuti dialect yang dipilih |
| `DB_FILE` | `./data/sandbox.db` | Path file database (hanya bila `DB_TYPE=sqlite`) |

Pada mode `sqlite`, parameter `DB_HOST`, `DB_PORT`, `DB_USER`, dan `DB_PASSWORD` diabaikan. Seluruh nilai hasil input ditulis ke `config/<--config>` (file di-`init` lebih dulu bila belum ada).

## Pilihan Scope (Scope Selection)

| Pilihan | Scope | Yang di-generate |
|---------|-------|------------------|
| `1` | REST API only | Backend saja (SDF → database, RDF, endpoint) |
| `2` (default) | REST API + Frontend app | Backend ditambah frontend app (UDF → HTML/JS/CSS) |

Tidak tersedia opsi "frontend only" karena pipeline frontend bergantung pada artefak yang dihasilkan pipeline REST API (RDF sebagai input migrate ke UDF, dan runtime UI fetch ke REST API server).

## Pipeline Eksekusi (Execution Pipeline)

Jalannya Pipeline REST API mengikuti alur orkestrasi command yang sudah ada:

| Fase | Aksi |
|------|------|
| `[1/4]` | Config, license, dan validasi database: tulis env (`init` bila perlu) → `validate --auto-create-db` → `config set-default` |
| `[2/4]` | Schema migrate: `schema migrate --path=<schema>` (`--drop=true` bila `--overwrite`) |
| `[3/4]` | Payload RDF: `payload generate` per tabel |
| `[3b/4]` | FK expansion: `payload sync --expand-fk` untuk tabel yang memiliki foreign key |
| `[4/4]` | REST endpoints: `endpoint create` per tabel |

Setelah pipeline backend, command menulis launcher `server-start.bat` (Windows) atau `server-start.sh` (Linux/macOS) di folder kerja untuk start REST API secara manual.

Bila scope mencakup frontend, pipeline berikut dijalankan setelahnya:

| Fase | Aksi |
|------|------|
| `[F1/3]` | Migrate RDF → UDF: `payload migrate` per tabel (tabel pure-parent dilewati karena ditemukan secara otomatis via JOIN dari child) |
| `[F2/3]` | Activate: `restforge-designer activate` |
| `[F3/3]` | Generate: `restforge-designer generate` aplikasi frontend |

Di akhir, command menawarkan menjalankan runtime server (membuka window baru dan menunggu health check) dan frontend app (`npx serve`) di window terpisah.

## Mode Sync dan Overwrite

| Mode | Pemicu | Perilaku schema migrate | Konfirmasi |
|------|---------|-------------------------|------------|
| Sync (default) | tanpa `--overwrite` | `CREATE TABLE IF NOT EXISTS` (tabel yang sudah ada dilewati) | `Y/n` |
| Overwrite | `--overwrite` | `DROP TABLE` lalu re-create (data hilang permanen) | `y/N` |

Pada mode `--overwrite`, file lama (RDF, endpoint, frontend) diarsipkan lebih dulu sebelum ditimpa (mis. `payload/visitors.json.archive.001`). Operasi ini bersifat destruktif dan tidak dapat dibatalkan.

## Dependency restforge-designer

Scope frontend membutuhkan binary `restforge-designer` ter-install. Pada tahap preflight, command menjalankan `restforge-designer --version`. Bila binary tidak ditemukan, eksekusi dihentikan disertai instruksi download dari `https://restforge.dev/download.html`.

## Contoh (Examples)

```bat
:: Generate REST API + frontend app (default scope)
npx restforge fast-track --project=visitors-app --schema-path=./schema

:: Tentukan file config env target secara eksplisit
npx restforge fast-track --project=visitors-app --schema-path=./schema --config=db-connection.env

:: Mode destruktif: drop table dan regenerate
npx restforge fast-track --project=visitors-app --schema-path=./schema --overwrite
```

## Catatan (Notes)

- Command bersifat interaktif (prompt LICENSE, database, scope, konfirmasi). Pada lingkungan non-interaktif (CI/CD), seluruh input perlu di-pipe ke stdin.
- Pembukaan window service baru (runtime server dan frontend) memakai `cmd start` sehingga spesifik Windows. Launcher `server-start.bat`/`.sh` tetap ditulis untuk start manual lintas OS.
- Berbeda dengan `schema introspect`, scope backend `fast-track` menerima `DB_TYPE=sqlite`.
- `fast-track` tidak memuat logic generate baru. Seluruh artefak dihasilkan oleh command yang diorkestrasi: `init`, `validate`, `config set-default`, `schema migrate`, `payload generate`, `payload sync`, `endpoint create`, `payload migrate`, serta `restforge-designer activate` dan `generate`.

---

**Lihat juga**: [`commands/`](./) · [`schema/migrate`](./schema/migrate.md) · [`endpoint/create`](./endpoint/create.md) · [`restforge-frontend/generate`](../restforge-frontend/generate.md) · [`conventions`](./conventions.md) · [`README`](../README.md)
