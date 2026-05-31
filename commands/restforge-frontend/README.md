# `restforge-designer` Commands

> Daftar lengkap verb CLI binary `restforge-designer` (frontend code generator).

`restforge-designer` adalah binary **terpisah** dari `restforge` backend. Pattern command berbeda: tidak ada `<resource>` di antara binary name dan verb.

```
restforge-designer <verb> [options]
```

Binary file di filesystem:
- Nama lengkap: `restforge-designer.exe` (Windows) / `restforge-designer` (Linux/macOS)
- Alias pendek: `rfd.exe` / `rfd`

Dokumentasi ini memakai nama lengkap `restforge-designer` untuk kejelasan. Pengguna yang lebih suka alias pendek dapat mengganti seluruh `restforge-designer` dengan `rfd` di contoh-contoh berikut.

## Daftar Verb (Verb List)

| Verb | Tujuan | Halaman |
|------|--------|---------|
| `init` | Scaffold project baru dari plugin | [`init.md`](./init.md) |
| `validate` | Validasi UDF (payload JSON) terhadap schema plugin | [`validate.md`](./validate.md) |
| `preview` | Preview daftar file hasil generate tanpa write ke disk | [`preview.md`](./preview.md) |
| `generate` | Generate aplikasi frontend dari UDF | [`generate.md`](./generate.md) |
| `activate` | Aktivasi license, simpan key ke encrypted user storage | [`activate.md`](./activate.md) |
| `deactivate` | Hapus license dari encrypted user storage | [`deactivate.md`](./deactivate.md) |
| `license status` | Tampilkan status license aktif | [`license-status.md`](./license-status.md) |
| `license migrate` | Migrasi license key dari `.env` ke encrypted storage | [`license-migrate.md`](./license-migrate.md) |
| `plugins list` | Daftar plugin yang ter-install | [`plugins-list.md`](./plugins-list.md) |
| `plugins inspect` | Tampilkan metadata satu plugin | [`plugins-inspect.md`](./plugins-inspect.md) |
| `plugins scaffold` | Scaffold folder plugin custom baru | [`plugins-scaffold.md`](./plugins-scaffold.md) |

## Global Options

Beberapa flag berlaku untuk semua verb (dapat ditulis sebelum atau setelah verb):

| Flag | Tipe | Keterangan |
|------|------|-----------|
| `--plugins-dir <PATH>` | string | Path ke folder plugins. Default: auto-detect. Setara env var `RESTFORGE_DESIGNER_PLUGINS_DIR` |
| `--license <KEY>` | string | License key format `XXXX-XXXX-XXXX-XXXX`. Override storage, `.env`, env var per-invocation |
| `-h`, `--help` | flag | Tampilkan help untuk verb atau binary |
| `-V`, `--version` | flag | Tampilkan versi binary (hanya di level binary, bukan per verb) |

## Konvensi (Conventions)

Detail konvensi flag, separator (`=` vs space), dan precedence dijelaskan di [`conventions.md`](./conventions.md).

## Alur Workflow Umum

Alur dari setup hingga produksi aplikasi frontend:

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ activate │ → │  init    │ → │ validate │ → │ preview  │ → │ generate │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
   one-time      one-time      iterative      iterative      final
   per machine    per project   per change     per change     per change
```

| Tahap | Command | Frekuensi |
|-------|---------|-----------|
| Setup license | `restforge-designer activate --key XXXX-XXXX-XXXX-XXXX` | Sekali per mesin |
| Scaffold project | `restforge-designer init --plugin vanilla-js-basic --output ./myapp` | Sekali per project |
| Edit UDF | (manual) | Iteratif |
| Validate sebelum generate | `restforge-designer validate --payload payload/app.json` | Sebelum generate |
| Preview file output | `restforge-designer preview --payload payload/app.json` | Optional, sebelum generate |
| Generate aplikasi | `restforge-designer generate --payload payload/app.json --output ./myapp --overwrite` | Setelah setiap perubahan UDF |

## Alur Migrasi dari Backend

Jika sudah punya RDF (backend payload), bisa di-convert menjadi UDF sebagai starting point. **Konversi dilakukan di sisi backend** dengan command `npx restforge payload migrate` (bukan di binary designer).

```
┌──────────────┐   ┌──────────────────────┐   ┌──────────┐   ┌──────────┐
│ payload/     │ → │ npx restforge        │ → │ validate │ → │ generate │
│ backend/...  │   │ payload migrate      │   │          │   │          │
│ visitors.json│   └──────────────────────┘   └──────────┘   └──────────┘
└──────────────┘
```

```bash
npx restforge payload migrate --name=visitors.json --project=myapp --output=..\frontend\payload
```

Detail di [`commands/restforge-backend/payload/migrate.md`](../restforge-backend/payload/migrate.md).

## Hubungan dengan UDF Catalog

Halaman ini fokus pada **command CLI**. Untuk struktur dan field UDF (UI Definition File) yang menjadi input verb `validate`, `preview`, dan `generate`, lihat [`catalogs/udf/`](../../catalogs/udf/).

| Kebutuhan | Lokasi |
|-----------|--------|
| Flag dan opsi command | Halaman per verb di folder ini |
| Struktur UDF (`appConfig`, `pages`, `fields`, dll.) | [`catalogs/udf/`](../../catalogs/udf/) |
| Aturan validasi UDF | [`catalogs/udf/validation-rules.md`](../../catalogs/udf/validation-rules.md) |
| Contoh UDF lengkap | [`catalogs/udf/complete-examples.md`](../../catalogs/udf/complete-examples.md) |

---

**Lihat juga**: [`commands/`](../) · [`catalogs/udf/`](../../catalogs/udf/) · [`README`](../../README.md)
