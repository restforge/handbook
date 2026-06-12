# `restforge-designer plugins scaffold`

> Scaffold folder plugin custom baru dengan struktur file standar.

## Sintaks

```bash
restforge-designer plugins scaffold --id <ID> --output <OUTPUT> [OPTIONS]
```

## Flag

| Flag | Tipe | Wajib | Keterangan |
|------|------|:-----:|-----------|
| `--id <ID>` | string | ✓ | Plugin ID baru (kebab-case direkomendasikan) |
| `-o`, `--output <OUTPUT>` | path | ✓ | Folder output untuk scaffold plugin |
| `--plugins-dir <PATH>` | string | ✗ | Override folder plugins (env: `RESTFORGE_DESIGNER_PLUGINS_DIR`) |
| `--license <KEY>` | string | ✗ | License key override per-invocation |

## Contoh Penggunaan

### Scaffold Plugin Custom Sederhana

```bash
restforge-designer plugins scaffold --id=my-custom-plugin --output=./plugins/my-custom-plugin
```

Output:

```
╭─ Plugin Scaffolded ────────────────────────────────╮
│ Plugin ID : my-custom-plugin                       │
│ Output    : ./plugins/my-custom-plugin             │
│                                                    │
│ Files created:                                     │
│   plugin.json                                      │
│   templates/                                       │
│   assets/                                          │
│   README.md                                        │
╰────────────────────────────────────────────────────╯

Next steps:
  1. Edit plugin.json with metadata
  2. Add templates to templates/ (Jinja2 syntax)
  3. Test with: restforge-designer plugins inspect --plugin=my-custom-plugin --plugins-dir=./plugins
```

### Scaffold ke Plugins Directory Default

```bash
restforge-designer plugins scaffold --id=my-plugin --output=$(restforge-designer plugins list --plugins-dir | head -1)/my-plugin
```

> **Catatan:** Lebih aman scaffold ke folder development terpisah dulu, baru deploy ke plugins directory default setelah testing.

## Behavior

Verb `plugins scaffold`:

1. **Validasi ID** plugin (format identifier valid)
2. **Cek folder output**:
   - Jika sudah ada → error
   - Jika tidak ada → buat folder
3. **Tulis file struktur standar**:
   - `plugin.json` — metadata plugin (ID, version, capabilities)
   - `templates/` — folder template Jinja2 untuk file yang di-generate
   - `assets/` — folder aset statis (CSS, JS vendor, dll.)
   - `README.md` — dokumentasi plugin
4. **Tampilkan ringkasan** + next steps

## Struktur Plugin yang Di-scaffold

```
my-custom-plugin/
├── plugin.json              # Metadata plugin
├── README.md                # Dokumentasi plugin
├── templates/               # Template Jinja2
│   ├── index.html.j2        # Template homepage
│   ├── page.html.j2         # Template per-page
│   ├── common.js.j2         # Template shared JS
│   ├── page.js.j2           # Template per-page JS
│   ├── app-start.bat.j2     # Template script start
│   └── ...
└── assets/                  # File static yang di-copy ke output
    ├── css/
    │   └── global.css
    └── vendor/
        └── (libraries)
```

Struktur persis bergantung pada versi `restforge-designer`. Jalankan `plugins inspect` setelah scaffold untuk verifikasi metadata terbaca.

## `plugin.json` — Metadata Plugin

File ini berisi:

| Field | Tipe | Keterangan |
|-------|------|-----------|
| `id` | string | Plugin ID (harus match `--id` dan nama folder) |
| `name` | string | Nama display plugin |
| `version` | string | Versi semver |
| `author` | string | Pengembang |
| `description` | string | Deskripsi singkat |
| `auth_required` | boolean | Apakah plugin butuh konfigurasi auth |
| `capabilities` | array | Fitur UDF yang didukung |
| `field_types` | array | Tipe field yang valid |
| `page_types` | array | Tipe page yang valid |

Edit `plugin.json` sesuai kebutuhan plugin.

## Template Jinja2

Template di `templates/` memakai sintaks Jinja2 untuk render file output. Variable yang tersedia:

| Variable | Sumber |
|----------|--------|
| `appConfig` | Field `appConfig` di UDF |
| `pages` | Array `pages` di UDF |
| `page` | Per-page rendering: page object aktif |
| `field` | Per-field rendering: field object aktif |
| `navigation` | Field `navigation` di UDF (jika ada) |
| `homepage` | Field `homepage` di UDF (jika ada) |

Contoh template `app-start.bat.j2`:

```jinja2
@echo off
echo Starting application...
echo Open http://localhost:{{ appConfig.port | default(8080) }}/index.html in your browser
npx serve . -l {{ appConfig.port | default(8080) }}
```

## Validasi Plugin Custom

Setelah scaffold + edit metadata, verifikasi:

```bash
restforge-designer plugins inspect --plugin=my-custom-plugin --plugins-dir=./plugins
```

Jika metadata terbaca dengan benar, plugin siap dipakai di `init` atau `generate`:

```bash
restforge-designer init --plugin=my-custom-plugin --output=./test-app --plugins-dir=./plugins
```

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| `0` | Scaffold sukses |
| `1` | ID invalid, folder output sudah ada, atau IO error |

## Workflow Pengembangan Plugin

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ scaffold │ → │   edit   │ → │ inspect  │ → │ generate │
│  plugin  │   │ metadata │   │ (verify) │   │ (test)   │
└──────────┘   │ + template│   └──────────┘   └──────────┘
               └──────────┘
```

| Tahap | Command |
|-------|---------|
| 1. Scaffold | `restforge-designer plugins scaffold --id=my-plugin --output=./plugins/my-plugin` |
| 2. Edit `plugin.json` dan template | (manual) |
| 3. Verifikasi metadata | `restforge-designer plugins inspect --plugin=my-plugin --plugins-dir=./plugins` |
| 4. Test generate | `restforge-designer init --plugin=my-plugin --output=./test --plugins-dir=./plugins` lalu `restforge-designer generate ...` |

## Use Case

| Skenario | Behavior |
|----------|----------|
| Mau bikin plugin custom untuk design system internal | `plugins scaffold` lalu edit template |
| Fork plugin built-in untuk modifikasi | Manual copy plugin built-in, bukan `scaffold` |
| Plugin untuk framework spesifik (Vue, React) | `plugins scaffold` baru dari awal, replace template Jinja2 |

---

**Lihat juga**: [`README`](./README.md) · [`plugins-list.md`](./plugins-list.md) · [`plugins-inspect.md`](./plugins-inspect.md) · [`catalogs/udf/`](../../catalogs/udf/)
