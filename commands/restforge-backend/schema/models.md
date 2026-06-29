# `schema models`

> Menampilkan daftar schema models dengan ringkasan struktur (fields, primary key, indexes, uniques, relations).

## Pattern

```
npx restforge schema models --schema-path=<PATH>
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--schema-path <PATH>` | Ya | - | Path folder schema (mis. `./schema`) |

## Contoh

```bat
npx restforge schema models --schema-path=./schema
```

---

**Lihat juga**: [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
