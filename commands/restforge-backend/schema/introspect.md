# `schema introspect`

> Reverse-engineer database existing menjadi file schema definition (dbschema-kit factory function).

## Pattern

```
npx restforge schema introspect --config=<FILE> --output=<PATH> [options]
```

## Flag Wajib (Required)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--config <FILE>` | - | File config database |
| `--output <PATH>` | - | Folder atau file output |

## Flag Opsional (Optional)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--table <NAME>` | semua | Hanya satu tabel (unqualified atau qualified) |
| `--schema <LIST>` | default schema | Filter schema (comma-separated) |
| `--all-schemas` | `false` | Auto-detect seluruh user schemas |
| `--force` | `false` | Timpa file existing |
| `--dry-run` | `false` | Print ke stdout tanpa menulis file |

## Contoh (Examples)

```bat
:: Introspect seluruh tabel
npx restforge schema introspect --config=db.env --output=./schema

:: Introspect tabel spesifik dengan dry-run
npx restforge schema introspect --config=db.env --output=./schema --table=users --dry-run

:: Introspect seluruh user schemas
npx restforge schema introspect --config=db.env --output=./schema --all-schemas --force
```

---

**Lihat juga**: [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
