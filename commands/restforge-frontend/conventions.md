# Konvensi CLI `restforge-designer`

> Aturan penulisan command dan opsi untuk binary `restforge-designer`.

## Pattern Umum

```
restforge-designer <verb> [options]
```

Tidak seperti `restforge` (backend) yang memakai pattern `<resource> <verb>`, binary `restforge-designer` memakai pattern **verb-only** di level top. Beberapa verb (`license`, `plugins`) memiliki subcommand di level kedua:

```
restforge-designer license <subverb> [options]
restforge-designer plugins <subverb> [options]
```

## Penamaan Verb

| Verb | Tipe | Catatan |
|------|------|--------|
| `init` | Top-level | Tidak konflik dengan resource `restforge init` (binary berbeda) |
| `validate` | Top-level | Validasi UDF (bukan SDF/RDF seperti di backend) |
| `preview` | Top-level | Khusus designer (tidak ada di backend) |
| `generate` | Top-level | Khusus designer (tidak ada di backend dengan nama persis sama) |
| `activate` | Top-level | Setup license |
| `deactivate` | Top-level | Hapus license |
| `license` | Parent | Berisi subverb `status` dan `migrate` |
| `plugins` | Parent | Berisi subverb `list`, `inspect`, `scaffold` |

## Format Flag

### Long-Flag Wajib untuk Konfigurasi

Konfigurasi command memakai long-flag dengan double-dash:

```bash
restforge-designer generate --payload=payload/visitors.json --output=./apps/visitors-app --overwrite
```

### Short-Flag yang Tersedia

Beberapa flag memiliki short-form satu huruf:

| Long-Flag | Short-Flag | Verb yang Punya |
|-----------|-----------|----------------|
| `--payload <FILE>` | `-p` | `validate`, `preview`, `generate` |
| `--output <PATH>` | `-o` | `init`, `generate`, `plugins scaffold` |
| `--yes` | `-y` | `deactivate` |
| `--help` | `-h` | Semua verb |
| `--version` | `-V` | Hanya di binary root |

### Separator: Spasi atau `=`

Kedua bentuk valid:

```bash
restforge-designer generate --payload payload/visitors.json --output ./apps/visitors-app
restforge-designer generate --payload=payload/visitors.json --output=./apps/visitors-app
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

Dengan single-line, contoh command dapat di-copy-paste langsung ke shell apapun tanpa modifikasi. Pengguna yang ingin memecah jadi multi-line untuk script lokal dapat menambahkan karakter line continuation sesuai shell pilihan.

### Boolean Flag

Boolean flag ditulis bare (tanpa value):

```bash
restforge-designer generate --payload=payload/visitors.json --output=./apps/visitors-app --overwrite
restforge-designer activate --key=ABCD-1234-EFGH-5678 --no-validate --force
```

Tidak ada bentuk `--overwrite=true` atau `--overwrite=false` di CLI clap-based ini. Untuk menonaktifkan, cukup hilangkan flag.

## Precedence Resolusi License Key

Saat verb apapun butuh license key, resolusi berurutan:

| Prioritas | Sumber | Cara Set |
|-----------|--------|----------|
| 1 | CLI flag `--license <KEY>` | Per-invocation override |
| 2 | Encrypted storage | `restforge-designer activate --key XXXX-XXXX-XXXX-XXXX` |
| 3 | File `.env` field `LICENSE=` | Manual edit `.env` di working directory |
| 4 | OS env var `RESTFORGE_DESIGNER_LICENSE_KEY` | `set` / `export` per shell |

Yang ketemu duluan dipakai. Verb `init` tidak memerlukan license karena hanya copy file dari plugin scaffold. Verb lain (`validate`, `preview`, `generate`, `plugins inspect`, dst.) memerlukan license aktif.

> **Catatan:** Konversi RDF → UDF (`payload migrate`) bukan dilakukan di binary `restforge-designer`, melainkan di binary backend `restforge`. Detail: [`commands/restforge-backend/payload/migrate.md`](../restforge-backend/payload/migrate.md).

## Format License Key

Pattern wajib: `XXXX-XXXX-XXXX-XXXX` (4 segmen, masing-masing 4 karakter uppercase A-Z atau 0-9, dipisah dash).

| Valid | Invalid | Alasan |
|-------|---------|--------|
| `ABCD-1234-EFGH-5678` | `abcd-1234-efgh-5678` | Harus uppercase |
| `0000-1111-2222-3333` | `ABCD12345EFGH5678` | Harus ada dash |
| `XXXX-YYYY-ZZZZ-WWWW` | `ABC-1234-EFGH-5678` | Segmen pertama hanya 3 char |

CLI otomatis melakukan `.trim().toUpperCase()` sebelum validasi, jadi spasi sekitar dan lowercase di input akan dinormalisasi.

## Env Var yang Dikenali

| Env Var | Tujuan |
|---------|--------|
| `RESTFORGE_DESIGNER_LICENSE_KEY` | License key (priority 4 di resolusi) |
| `RESTFORGE_DESIGNER_PLUGINS_DIR` | Override path folder plugins (default: auto-detect) |
| `RESTFORGE_DESIGNER_CONFIG_DIR` | Override user config dir (default: `~/.restforge-designer/`). Khusus test/CI |
| `RESTFORGE_DESIGNER_ROOT_DIR` | Override root dir tempat `.env` dibaca. Khusus test/CI |

## Penanganan Exit Code

| Exit Code | Arti |
|-----------|------|
| `0` | Success |
| `1` | Failure (validation error, file not found, license invalid, dll.) |

Verb yang menulis ke disk (`init`, `generate`) akan exit `1` jika output sudah ada dan flag `--overwrite` tidak dipakai.

## Lokasi User Storage

Encrypted user storage dan cache:

| OS | Lokasi |
|----|--------|
| Windows | `C:\Users\<user>\.restforge-designer\` (via Windows Credential Manager untuk keyring, dengan fallback encrypted file) |
| macOS | `~/.restforge-designer/` (via Keychain) |
| Linux | `~/.restforge-designer/` (via Secret Service) |

Tidak per-project. Sekali `activate`, license berlaku global untuk semua project di user account tersebut. Lihat [`activate.md`](./activate.md) untuk detail.

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
