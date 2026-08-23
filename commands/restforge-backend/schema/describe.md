# `schema describe`

> Menampilkan detail struktur satu tabel: kolom, primary key, foreign keys, dan indexes.

## Pattern

```
npx restforge schema describe --config=<FILE> --table=<NAME> [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--config <FILE>` | Ya | - | File config database |
| `--table <NAME>` | Ya | - | Nama tabel (`users` atau qualified `public.users`) |
| `--include-foreign-keys <BOOL>` | Tidak | `true` | Sertakan informasi foreign key |
| `--include-indexes <BOOL>` | Tidak | `true` | Sertakan informasi indexes |
| `--pretty <BOOL>` | Tidak | `true` | Pretty-print output JSON |

## Contoh

```bat
npx restforge schema describe --config=db.env --table=users
npx restforge schema describe --config=db.env --table=public.orders --include-indexes=false
```

---

**Lihat juga**: [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
