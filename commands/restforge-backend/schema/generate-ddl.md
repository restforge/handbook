# `schema generate-ddl`

> Generate DDL SQL (CREATE TABLE, CREATE INDEX, opsional DROP) untuk seluruh schema models.

## Pattern

```
npx restforge schema generate-ddl --schema-path=<PATH> --dialect=<TYPE> [options]
```

## Flag Wajib

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--schema-path <PATH>` | - | Path file atau folder schema |
| `--dialect <TYPE>` | - | Target SQL dialect (`postgres`, `mysql`, `oracle`, `sqlite`) |

## Flag Opsional

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--output <FILE>` | stdout | File output untuk DDL hasil generate |
| `--drop <BOOL>` | `false` | Sertakan statement DROP TABLE sebelum CREATE |

## Contoh

```bat
npx restforge schema generate-ddl --schema-path=./schema --dialect=postgres --output=migration.sql
npx restforge schema generate-ddl --schema-path=./schema --dialect=mysql --drop=true
```

---

**Lihat juga**: [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
