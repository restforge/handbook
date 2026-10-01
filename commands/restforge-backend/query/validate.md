# `query validate`

> Memvalidasi statement SELECT/CTE terhadap database live menggunakan EXPLAIN (read-only, tidak mengeksekusi data modification).

## Pattern

```
npx restforge query validate --sql="<SQL>" [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--sql "<SQL>"` | Ya | - | Statement SQL untuk divalidasi (hanya SELECT atau WITH) |
| `--config <FILE>` | Tidak | default terdaftar | File config database. Bila tidak diisi, CLI jatuh ke default yang tercatat di `.restforge/defaults.json` |
| `--pretty <BOOL>` | Tidak | `true` | Pretty-print output JSON |

Default config diatur lewat [`config set-default`](../config/set-default.md), sehingga nama file config tidak perlu diulang di setiap panggilan. Bila `--config` dikosongkan dan tidak ada default yang tercatat, CLI berhenti dengan pesan bahwa config wajib diisi.

## Contoh

```bat
npx restforge query validate --config=db.env --sql="SELECT * FROM users WHERE id = 1"
```

---

**Lihat juga**: [`query/`](./) · [`commands/`](../) · [`README`](../../../README.md)
