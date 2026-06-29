# `query validate`

> Memvalidasi statement SELECT/CTE terhadap database live menggunakan EXPLAIN (read-only, tidak mengeksekusi data modification).

## Pattern

```
npx restforge query validate --config=<FILE> --sql="<SQL>" [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--config <FILE>` | Ya | - | File config database |
| `--sql "<SQL>"` | Ya | - | Statement SQL untuk divalidasi (hanya SELECT atau WITH) |
| `--pretty <BOOL>` | Tidak | `true` | Pretty-print output JSON |

## Contoh

```bat
npx restforge query validate --config=db.env --sql="SELECT * FROM users WHERE id = 1"
```

---

**Lihat juga**: [`query/`](./) · [`commands/`](../) · [`README`](../../../README.md)
