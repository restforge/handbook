# `init` (global verb)

> Generate sample starter files (config database, payload contoh, query template SQL) di working directory saat ini. Berguna untuk inisialisasi project baru.

Verb `init` adalah verb global yang tidak terikat pada resource tertentu, sehingga pemanggilannya cukup `npx restforge init` tanpa prefix resource.

## Pattern

```
npx restforge init [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--force` / `-f` | Tidak | `false` | Timpa file existing tanpa konfirmasi |

## File yang Di-generate

| Path | Keterangan |
|------|-----------|
| `config/db-connection.env` | Template database connection config |
| `payload/samples.json` | Sample payload untuk tabel `contact` |
| `payload/query/samples-datatables.sql` | Sample SQL query untuk datatables endpoint |

## Contoh (Examples)

```bat
npx restforge init
npx restforge init --force
npx restforge init -f
```

---

**Lihat juga**: [`commands/`](./) · [`conventions`](./conventions.md) · [`README`](../README.md)
