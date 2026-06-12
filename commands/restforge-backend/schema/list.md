# `schema list`

> Menampilkan daftar seluruh tabel pada database secara live introspection.

## Pattern

```
npx restforge schema list --config=<FILE> [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--config <FILE>` | Ya | - | File config database |
| `--schema <NAME>` | Tidak | semua | Filter berdasarkan nama schema |
| `--include-system <BOOL>` | Tidak | `false` | Sertakan system schemas |
| `--pretty <BOOL>` | Tidak | `true` | Pretty-print output JSON |

## Contoh (Examples)

```bat
npx restforge schema list --config=db.env
npx restforge schema list --config=db.env --schema=public
```

---

**Lihat juga**: [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
