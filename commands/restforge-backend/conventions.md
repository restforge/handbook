# Konvensi CLI (CLI Convention)

> Aturan umum yang berlaku untuk semua perintah RESTForge.

## Konvensi Flag-Only (Flag-Only Convention)

Seluruh argument di RESTForge CLI menggunakan **flag eksplisit**. Pattern utama tetap `npx restforge <resource> <verb> [options]` di mana `<resource>` dan `<verb>` adalah positional wajib di posisi pertama dan kedua. Selain itu, seluruh argument lain dideklarasikan sebagai flag bernama (`--name=<VALUE>`).

### Aturan

1. **Resource dan verb wajib positional** — selalu di posisi pertama dan kedua setelah `restforge`, mis. `restforge schema validate`
2. **Seluruh argument lain wajib flag-only** — tidak ada positional argument tambahan setelah verb
   - Path filesystem: `--path=<PATH>`, `--output=<PATH>`, `--config=<FILE>`
   - Identifier non-path: `--project=<NAME>`, `--table=<NAME>`, `--dialect=<TYPE>`
   - Boolean option: `--dry-run`, `--force` (bare flag, setara `--flag=true`)

### Contoh Aplikasi

| Command | Pattern | Catatan |
|---|---|---|
| `schema init --path=<PATH>` | Flag-only | Path file scaffold target |
| `schema validate --path=<PATH>` | Flag-only | Path folder schema (wajib) |
| `schema models --path=<PATH>` | Flag-only | Path folder schema (wajib) |
| `schema generate-ddl --path=<PATH> --dialect=<TYPE>` | Flag-only | Multiple flag |
| `schema migrate --path=<PATH> --config=<FILE>` | Flag-only | Multiple flag dengan peran berbeda |
| `schema diff --path=<PATH> --config=<FILE>` | Flag-only | Read-only drift detection |
| `schema apply --path=<PATH> --config=<FILE>` | Flag-only | Apply incremental ALTER |
| `schema introspect --config=<FILE> --output=<PATH>` | Flag-only | Multiple path role dengan disambiguasi flag |
| `payload generate --config=<FILE>` | Flag-only | Tidak ada path argument |

### Rationale

Konsistensi flag-only memberi keuntungan berikut:

- **Eksplisit dan unambiguous** — peran setiap argument langsung jelas dari nama flag, tanpa perlu mengingat urutan positional
- **Konsisten lintas command** — user tidak perlu memikirkan command mana yang pakai positional dan mana yang pakai flag
- **Ergonomis untuk scripting** — order tidak relevan, mudah untuk template generation di CI/CD
- **Mudah di-extend** — penambahan flag baru tidak memerlukan keputusan positional vs flag

### Catatan Historis

Sebelumnya beberapa schema command menerima `<path>` sebagai positional argument (mis. `schema validate ./schema`). Sejak versi flag-only refactor, syntax positional dihapus sepenuhnya (hard-break, tanpa backward compatibility). Pemanggilan dengan syntax lama akan menghasilkan error parser `Missing required flag: --path`.

## Global Options

Flag berikut berlaku pada level binary `restforge`:

| Flag | Keterangan |
|------|-----------|
| `--help` / `-h` | Tampilkan help message |
| `--version` / `-v` | Tampilkan versi package |

Help spesifik per command dapat diakses dengan menjalankan command tersebut diikuti `--help`, contoh: `npx restforge schema introspect --help`.

## Catatan Penggunaan (Usage Notes)

- **Resolusi `--config`**: jika `--config` tidak disediakan eksplisit, command yang menerima parameter ini (resource `payload`, `schema`, `query`) akan fallback ke default config dari `.restforge/defaults.json`. Saat fallback aktif, warning di-print ke stderr agar pengguna mengetahui config mana yang dipakai. Parameter `--config` eksplisit selalu memiliki prioritas lebih tinggi daripada default.
- **Flag `--force`** bersifat **destructive** dan harus digunakan dengan hati-hati karena file existing akan tertimpa tanpa konfirmasi.
- **Command read-only** yang aman dijalankan kapan saja: `validate`, `payload diff`, `schema list`, `schema describe`, `schema validate`, `schema models`, `schema generate-ddl` (tanpa `--out`), `query validate`, seluruh `catalog *`, serta `config get-default` dan `config list`. Command ini tidak melakukan modifikasi pada filesystem maupun database.
- **Exit code**: `0` untuk sukses, `≠ 0` untuk failure (validation error, file write error, license invalid, database connection error, dll.). Konsisten untuk integrasi scripting dan CI/CD.
- **Boolean flag format**: `--flag=true|false` (eksplisit) atau short form `--flag` (implicit true). Contoh: `--cluster` setara `--cluster=true`, sedangkan `--pretty=false` digunakan untuk menonaktifkan default boolean.
- **Invocation alternatif**: `npx restforge ...` ekuivalen dengan `restforge ...` jika binary sudah ter-install secara global (`npm install -g @restforgejs/platform`). Penggunaan `npx` disarankan untuk local install dalam project.
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

Dengan single-line, contoh command dapat di-copy-paste langsung ke shell apapun tanpa modifikasi. Pengguna yang ingin memecah jadi multi-line untuk script lokal dapat menambahkan karakter line continuation sesuai shell pilihan.

---

**Lihat juga**: [`commands/`](./) · [`README`](../README.md)
