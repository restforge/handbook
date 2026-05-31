# `schema models`

> Menampilkan daftar schema models dengan ringkasan struktur (fields, primary key, indexes, uniques, relations).

## Pattern

```
npx restforge schema models --path=<PATH>
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--path <PATH>` | Ya | - | Path folder schema (mis. `./schema`) |

## Contoh (Examples)

```bat
npx restforge schema models --path=./schema
```

---

**Lihat juga**: [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
