# Resource `config`

> Resource untuk mengelola config schema, template, dan default config per working directory.

## Sub-command (6)

| Verb | Fungsi |
|------|--------|
| [`schema`](./schema.md) | Output JSON schema parameter `db-connection.env` |
| [`template`](./template.md) | Output raw template `db-connection.env` |
| [`set-default`](./set-default.md) | Tetapkan file config default untuk working directory |
| [`get-default`](./get-default.md) | Tampilkan file config default aktif |
| [`clear-default`](./clear-default.md) | Hapus default config untuk working directory |
| [`list`](./list.md) | Daftar file `.env` di cwd dan folder `config/` |

## Konvensi Khusus (Specific Convention)

Default config disimpan di `.restforge/defaults.json` per working directory. Command yang menerima `--config` (payload, schema, query) akan fallback ke default ini jika `--config` tidak disediakan eksplisit.

---

**Lihat juga**: [`commands/`](../) · [`README`](../../../README.md)
