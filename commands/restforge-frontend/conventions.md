# Konvensi CLI `npx restforge-designer`

> Aturan penulisan command dan opsi untuk binary `restforge-designer`.

Bentuk pemanggilan utama adalah `npx restforge-designer` karena binary di-bundle dalam package `@restforgejs/platform` (tersedia begitu project dibuat via `npx create-restforge-app` atau setelah `npm install @restforgejs/platform`). Bentuk standalone `restforge-designer` (binary file `restforge-designer.exe` / `restforge-designer`) atau alias pendek `rfd` tetap valid bila diinstall manual.

## Pattern Umum

```
npx restforge-designer <verb> [options]
```

Tidak seperti `restforge` (backend) yang memakai pattern `<resource> <verb>`, binary `restforge-designer` memakai pattern **verb-only** di level top. Verb `plugins` memiliki subcommand di level kedua:

```
npx restforge-designer plugins <subverb> [options]
```

## Penamaan Verb

| Verb | Tipe | Catatan |
|------|------|--------|
| `init` | Top-level | Tidak konflik dengan resource `restforge init` (binary berbeda) |
| `validate` | Top-level | Validasi UDF (bukan SDF/RDF seperti di backend) |
| `preview` | Top-level | Khusus designer (tidak ada di backend) |
| `generate` | Top-level | Khusus designer (tidak ada di backend dengan nama persis sama) |
| `catalog` | Top-level | Emit catalog UDF machine-readable |
| `plugins` | Parent | Berisi subverb `list`, `inspect`, `scaffold` |

## Format Flag

### Long-Flag Wajib untuk Konfigurasi

Konfigurasi command memakai long-flag dengan double-dash:

```bash
npx restforge-designer generate --payload=payload/visitors.json --output=./apps/visitors-app --overwrite
```

### Short-Flag yang Tersedia

Beberapa flag memiliki short-form satu huruf:

| Long-Flag | Short-Flag | Verb yang Punya |
|-----------|-----------|----------------|
| `--payload <FILE>` | `-p` | `validate`, `preview`, `generate` |
| `--output <PATH>` | `-o` | `init`, `generate`, `plugins scaffold` |
| `--help` | `-h` | Semua verb |
| `--version` | `-V` | Hanya di binary root |

### Separator: Spasi atau `=`

Kedua bentuk valid:

```bash
npx restforge-designer generate --payload payload/visitors.json --output ./apps/visitors-app
npx restforge-designer generate --payload=payload/visitors.json --output=./apps/visitors-app
```

Bentuk `=` direkomendasikan untuk konsistensi dengan dokumentasi backend (`restforge`) yang menggunakan `--flag=value` di seluruh handbook.

### Penulisan Single-Line di Dokumentasi

Semua contoh command di handbook ditulis dalam **satu baris** (tidak memakai line continuation seperti `\`, `^`, atau `` ` ``).

Alasan: line continuation berbeda per shell:

| Shell | Karakter Line Continuation |
|-------|---------------------------|
| Bash, Zsh, Git Bash | `\` (backslash) |
| Windows CMD | `^` (caret) |
| PowerShell | `` ` `` (backtick) |

Dengan single-line, contoh command dapat di-copy-paste langsung ke shell apa pun tanpa modifikasi. Pengguna yang ingin memecah jadi multi-line untuk script lokal dapat menambahkan karakter line continuation sesuai shell pilihan.

### Boolean Flag

Boolean flag ditulis bare (tanpa value):

```bash
npx restforge-designer generate --payload=payload/visitors.json --output=./apps/visitors-app --overwrite
```

Tidak ada bentuk `--overwrite=true` atau `--overwrite=false` di CLI clap-based ini. Untuk menonaktifkan, cukup hilangkan flag.

> **Catatan:** Konversi RDF → UDF (`payload migrate`) bukan dilakukan di binary `restforge-designer`, melainkan di binary backend `restforge`. Detail: [`commands/restforge-backend/payload/migrate.md`](../restforge-backend/payload/migrate.md).

## Env Var yang Dikenali

| Env Var | Tujuan |
|---------|--------|
| `RESTFORGE_DESIGNER_PLUGINS_DIR` | Override path folder plugins (default: auto-detect) |
| `RESTFORGE_DESIGNER_CONFIG_DIR` | Override user config dir (default: `~/.restforge-designer/`). Khusus test/CI |
| `RESTFORGE_DESIGNER_ROOT_DIR` | Override root dir tempat `.env` dibaca. Khusus test/CI |

## Penanganan Exit Code

| Exit Code | Arti |
|-----------|------|
| `0` | Success |
| `1` | Failure (validation error, file not found, dll.) |

Verb yang menulis ke disk (`init`, `generate`) akan exit `1` jika output sudah ada dan flag `--overwrite` tidak dipakai.

## Hubungan dengan Konvensi Backend

| Aspek | Backend (`restforge`) | Designer (`restforge-designer`) |
|-------|----------------------|--------------------------------|
| Pattern | `<resource> <verb>` | `<verb>` atau `<parent> <subverb>` |
| Positional argument tambahan | Tidak (flag-only) | Tidak (flag-only) |
| Boolean flag | `--flag` setara `--flag=true` | `--flag` bare (tidak ada `=true`/`=false`) |
| Separator | `--flag=value` direkomendasikan | Sama, `--flag=value` direkomendasikan |
| Long-flag wajib | Ya | Ya (tetapi short-flag tersedia untuk beberapa) |

---

**Lihat juga**: [`README`](./README.md) · [`commands/restforge-backend/conventions.md`](../restforge-backend/conventions.md) (konvensi backend) · [`README handbook`](../../README.md)
