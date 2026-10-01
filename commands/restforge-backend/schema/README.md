# Resource `schema`

> Resource untuk database schema management (definisi schema deklaratif, validasi, migrasi, introspeksi, drift detection, dan reference template). Memiliki sebelas sub-verb dengan pembagian: live introspection, code-driven definition, dan reference catalog.

## Sub-command (11)

| Verb | Fungsi |
|------|--------|
| [`list`](./list.md) | Daftar tabel di database (live introspection) |
| [`describe`](./describe.md) | Detail struktur satu tabel (live introspection) |
| [`init`](./init.md) | Shortcut untuk scaffold file schema definition via template `dummy` (delegasi ke `schema template`) |
| [`validate`](./validate.md) | Validasi schema definition files |
| [`models`](./models.md) | Daftar schema models dengan structural summary |
| [`generate-ddl`](./generate-ddl.md) | Generate DDL SQL dari schema definition |
| [`migrate`](./migrate.md) | Apply schema ke database (load → validate → DDL → execute) |
| [`introspect`](./introspect.md) | Reverse-engineer database menjadi schema definition |
| [`diff`](./diff.md) | Drift detection antara SDF dan struktur tabel database (read-only) |
| [`apply`](./apply.md) | Apply drift dari SDF ke database via ALTER TABLE incremental |
| [`template`](./template.md) | Browse, preview, dan generate template schema dari koleksi reference |

---

**Lihat juga**: [`commands/`](../) · [`README`](../../../README.md)
