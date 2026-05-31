# `schema generate-ddl`

> Generate DDL SQL (CREATE TABLE, CREATE INDEX, opsional DROP) untuk seluruh schema models.

## Pattern

```
npx restforge schema generate-ddl --path=<PATH> --dialect=<TYPE> [options]
```

## Flag Wajib (Required)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--path <PATH>` | - | Path file atau folder schema |
| `--dialect <TYPE>` | - | Target SQL dialect (`postgres`, `mysql`, `oracle`, `sqlite`) |

## Flag Opsional (Optional)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--out <FILE>` | stdout | File output untuk DDL hasil generate |
| `--drop <BOOL>` | `false` | Sertakan statement DROP TABLE sebelum CREATE |

## Contoh (Examples)

```bat
npx restforge schema generate-ddl --path=./schema --dialect=postgres --out=migration.sql
npx restforge schema generate-ddl --path=./schema --dialect=mysql --drop=true
```

---

**Lihat juga**: [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
