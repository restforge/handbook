# `restforge-designer plugins list`

> Daftar plugin yang ter-install di plugins directory.

## Sintaks

```bash
restforge-designer plugins list [OPTIONS]
```

## Flag

| Flag | Tipe | Default | Keterangan |
|------|------|---------|-----------|
| `--plugins-dir <PATH>` | string | auto-detect | Override folder plugins (env: `RESTFORGE_DESIGNER_PLUGINS_DIR`) |
| `--license <KEY>` | string | dari storage | License key override per-invocation |

## Contoh Penggunaan

### List Plugin Default

```bash
restforge-designer plugins list
```

Output umum:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Available Plugins                                                            │
├──────────────────┬─────────┬────────┬───────────────┬────────────────────────┤
│ ID               │ Version │ Auth   │ Library       │ Description            │
├──────────────────┼─────────┼────────┼───────────────┼────────────────────────┤
│ vanilla-js-basic │ 1.0.0   │ ✗      │ jQuery + CSS  │ Prototyping & simple   │
│ vanilla-js-auth  │ 1.0.0   │ ✓      │ jQuery + BS5  │ Production with JWT    │
└──────────────────┴─────────┴────────┴───────────────┴────────────────────────┘

Use `restforge-designer plugins inspect --plugin=<id>` for details.
```

### List dari Plugins Dir Custom

```bash
restforge-designer plugins list --plugins-dir=./my-plugins
```

Berguna untuk inspect plugin custom yang dikembangkan lokal sebelum di-install ke folder default.

## Behavior

Verb `plugins list`:

1. **Resolve plugins directory**:
   - `--plugins-dir` flag (jika dipakai)
   - Env var `RESTFORGE_DESIGNER_PLUGINS_DIR`
   - Auto-detect dari binary location
2. **Scan folder** untuk plugin yang valid (mengandung `plugin.json` atau metadata setara)
3. **Load metadata** masing-masing plugin
4. **Tampilkan tabel** ringkasan

## Plugin Bawaan (Built-in)

| Plugin ID | Auth | Library Utama | Cocok Untuk |
|-----------|:----:|---------------|-------------|
| `vanilla-js-basic` | ✗ | jQuery + Custom CSS | Prototyping, proof-of-concept, internal app tanpa auth |
| `vanilla-js-auth` | ✓ | jQuery + Bootstrap 5 + JWT | Production app dengan autentikasi |

Built-in plugin selalu tersedia tanpa perlu install eksplisit. Plugin custom di-install dengan copy folder ke plugins directory.

## Plugin Custom

Plugin custom dikenali jika folder di plugins directory memiliki struktur valid (file `plugin.json` + folder `templates/`, `assets/`, dll.). Lihat [`plugins-scaffold.md`](./plugins-scaffold.md) untuk membuat plugin custom baru.

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| `0` | List berhasil ditampilkan (termasuk jika kosong) |
| `1` | Plugins directory tidak ada atau tidak bisa dibaca |

## Lokasi Plugins Directory Default

Auto-detect bergantung pada cara binary di-install:

| Skenario | Lokasi Default |
|----------|----------------|
| Installer Windows | `C:\Program Files\RESTForge Designer\plugins\` |
| Cargo build (dev) | `<repo>\packages\designer\plugins\` atau auto-fallback ke built-in |
| Standalone binary | Folder yang sama dengan binary, atau auto-fallback |

Pakai `restforge-designer plugins list --plugins-dir=<custom-path>` untuk override.

## Use Case

| Skenario | Behavior |
|----------|----------|
| Cek plugin apa saja yang tersedia | `plugins list` |
| Develop plugin custom, lihat apakah sudah dikenali | `plugins list --plugins-dir=./my-plugins` |
| Pre-check sebelum `init` atau `generate` | `plugins list` untuk pastikan plugin target ter-install |

---

**Lihat juga**: [`README`](./README.md) · [`plugins-inspect.md`](./plugins-inspect.md) · [`plugins-scaffold.md`](./plugins-scaffold.md)
